Title: OpenStack Neutron DVR原理分析
Date: 2015-8-17
Category: OpenStack

##DVR 背景

DVR 是 Distributed Virtual Router 的缩写，目的是为了解决 OpenStack Neutron 部署的扩展性及单点故障问题。DVR 将原来集中由 Neutron Network Node 处理的流量负载，分发到各个 Compute Node 上

<img src="http://www.deepurple.info/images/2015-8-17/dvr-overview.png" alt="image" width="100%"/>

未使用DVR前的网络流量走向如上图左侧所示，可以明显看到，东西向和南北向的流量会集中到网络节点，这会使网络节点成为瓶颈，影响网络的可靠性与性能

启用 DVR 之后，网络流量走向如上图右侧所示

* 对于东西向的流量， 流量会直接在 Compute Node 之间传递，不再经过 Network Node；
* 对于南北向的流量，如果某个 Compute Node 的 VM 实例绑定了 Floating IP，则外网流量直接经过 Compute Node 传递，不经过 Network Node；
* 对于未绑定 Floating IP 的 VM 实例，外网流量依然需要经过 Network Node，经由 SNAT 处理后再传递

DVR 在 Neutron 中的实现涉及 L2 Agent 和 L3 Agent 两部分，以下分别进行原理和源码实现的分析

##DVR L2 Agent 原理

<img src="http://www.deepurple.info/images/2015-8-17/dvr-l2.png" alt="image" width="100%"/>

在上图中，一个 ping echo request 从红色网络中的 vm1 发送到绿色网络中的 vm2，这两个网络之间通过一个分布式路由器 r1 连接；r1 有两个接口（或者叫做“臂”），一个接口在红色网络的子网上，另一个在绿色网络的子网上； r1 在 CN1 和 CN2 上具有相同的 IP 地址和 MAC 地址

从 vm1 到 vm2 的 ping echo request 处理过程分为以下 6 步

1. 从 vm1 中发出以 vm2 ip 为目的 IP 的数据包，数据包的目的 MAC 地址为红色网络的默认网关 MAC，也即 r1 上红色子网的接口 MAC 地址（r1-red-mac），br-int 网桥收到该数据包后，将其转发给 r1

2. r1 的红色子网接口收到该数据包，然后对该 IP 数据包进行路由

3. 路由之后，r1 将这个数据包从绿色子网接口发送出去，这个数据包随后被 br-int 网桥交换到 br-tun 网桥，并且打上绿色网络的本地 VLAN Tag

4. CN1 节点的 br-tun 网桥用该节点被分配的一个唯一的 DVR MAC 地址来替换该数据包的源 MAC 地址，这个唯一的 DVR MAC 地址是由控制器分配给每个计算节点的。更改后的数据包通过 br-tun 网桥发送至 CN2 节点，在发送之前，br-tun 网桥将数据包的绿色网络 VLAN Tag剥除，并打上隧道的 VXLAN ID（green-vni）

5. CN2 节点上的 br-tun 网桥收到隧道发送过来的数据包，将隧道VXLAN ID （green-vni）去除，然后打上绿色网络的本地 VLAN Tag，随后将数据包发送至 br-int 网桥

6. 由于每个计算节点上的 L2 Agent 都可获得全局 DVR 路由器被分配到的特殊 MAC 地址，因此 CN2 节点上的 br-int 网桥可以识别到数据包的源 MAC 地址是一个独特的 DVR MAC 地址，随后 br-int 网桥将该数据包的 MAC 地址替换成绿色子网的 MAC 地址，并将数据包发送至 vm2

从 vm2 发送 ping response 给 vm1，过程与上面的类似

在数据包转发过程中，为了保证正确转发，需要提前填充相应的 ARP 表项，与 vm2 相对应的 ARP 表项是由运行在 CN1 节点上的 L3 Agent 填充到 CN1 节点的 DVR路由器 r1 中的，相应的 ARP 信息是由 L3 Agent 从 L3 Plugin 获得的；同样，与 vm1 相对应的 ARP 表项是由 CN2 节点上的 L3 Agent 按照从 L3 Plugin 获取的信息预填充到位于 CN2 节点上的 DVR 路由器 r1 中

##DVR L2 Agent 源码实现

在 neutron/plugins/ml2/drivers/openvswitch/agent/ovs\_neutron\_agent.py 中有如下 DVR 相关代码

        self.dvr_agent = ovs_dvr_neutron_agent.OVSDVRNeutronAgent(
            self.context,
            self.dvr_plugin_rpc,
            self.int_br,
            self.tun_br,
            self.bridge_mappings,
            self.phys_brs,
            self.int_ofports,
            self.phys_ofports,
            self.patch_int_ofport,
            self.patch_tun_ofport,
            self.conf.host,
            self.enable_tunneling,
            self.enable_distributed_routing)

        report_interval = self.conf.AGENT.report_interval
        if report_interval:
            heartbeat = loopingcall.FixedIntervalLoopingCall(
                self._report_state)
            heartbeat.start(interval=report_interval)

        if self.enable_tunneling:
            self.setup_tunnel_br()

        self.dvr_agent.setup_dvr_flows()

可见，该 agent 初始化时，调用了 ovs\_dvr\_neutron\_agent 用于创建 OVSDVRNeutronAgent 对象，并随后调用该对象的 setup\_dvr\_flows() 方法

查看 neutron/plugins/ml2/drivers/openvswitch/agent/ovs\_dvr\_neutron\_agent.py 源码中的 setup\_dvr\_flows() 方法

    def setup_dvr_flows(self):
        self.setup_dvr_flows_on_integ_br()
        self.setup_dvr_flows_on_tun_br()
        self.setup_dvr_flows_on_phys_br()
        self.setup_dvr_mac_flows_on_all_brs()

可见，setup\_dvr\_flows() 调用了四个其他函数，分别完成对 br-int / br-tun / phys-br 的流表初始化

以 setup\_dvr\_mac\_flows\_on\_all\_brs() 为例

    def setup_dvr_mac_flows_on_all_brs(self):
        if not self.in_distributed_mode():
            LOG.debug("Not in distributed mode, ignoring invocation "
                      "of get_dvr_mac_address_list() ")
            return
        dvr_macs = self.plugin_rpc.get_dvr_mac_address_list(self.context)
        LOG.debug("L2 Agent DVR: Received these MACs: %r", dvr_macs)
        for mac in dvr_macs:
            if mac['mac_address'] == self.dvr_mac_address:
                continue
            self._add_dvr_mac(mac['mac_address'])

首先通过 RPC 调用 get\_dvr\_mac\_address\_list()，获取统一分配的 DVR 专用 MAC 列表，然后调用 \_add\_dvr\_mac() 将获取到的 MAC 地址进行添加

而在 \_add\_dvr\_mac() 中，则分别调用了 \_add\_dvr\_mac\_for\_phys\_br() 和 \_add\_dvr\_mac\_for\_tun\_br()

    def _add_dvr_mac(self, mac):
        for physical_network in self.bridge_mappings:
            self._add_dvr_mac_for_phys_br(physical_network, mac)
        if self.enable_tunneling:
            self._add_dvr_mac_for_tun_br(mac)
        LOG.debug("Added DVR MAC flow for %s", mac)
        self.registered_dvr_macs.add(mac)

以 \_add\_dvr\_mac\_for\_tun\_br() 为例

    def _add_dvr_mac_for_tun_br(self, mac):
        self.int_br.add_dvr_mac_tun(mac=mac, port=self.patch_tun_ofport)
        self.tun_br.add_dvr_mac_tun(mac=mac, port=self.patch_int_ofport)

该函数其实是分别调用了 int\_br 和 tun\_br 的 add\_dvr\_mac\_tun() 方法，这两个方法的实现分别位于 neutron/plugins/ml2/drivers/openvswitch/agent/openflow/ovs\_ofctl 文件夹的 br\_int.py 和 br\_tun.py 中

查看 br\_int.py 中的 add\_dvr\_mac\_tun() 函数

    def add_dvr_mac_tun(self, mac, port):
        # Table LOCAL_SWITCHING will now sort DVR traffic from other
        # traffic depending on in_port
        self.install_goto(table_id=constants.LOCAL_SWITCHING,
                          priority=2,
                          in_port=port,
                          eth_src=mac,
                          dest_table_id=constants.DVR_TO_SRC_MAC)

可以看出，这条流表规则用于将 DVR 流量与其他流量进行分离，并将 DVR 流量提交到 DVR\_TO\_SRC\_MAC 这个流表中

查看 br\_tun.py 中的 add\_dvr\_mac\_tun() 函数

    def add_dvr_mac_tun(self, mac, port):
        # Table DVR_NOT_LEARN ensures unique dvr macs in the cloud
        # are not learnt, as they may result in flow explosions
        self.install_output(table_id=constants.DVR_NOT_LEARN,
                            priority=1,
                            eth_src=mac,
                            port=port)

可以看出，这条流表规则用于防止 DVR MAC 地址被学习

此外，在 ovs\_neutron\_agent.py 中的 \_bind\_distributed\_router\_interface\_port() 函数中，调用了如下方法

            self.int_br.install_dvr_to_src_mac(
                network_type=lvm.network_type,
                vlan_tag=vlan_to_use,
                gateway_mac=subnet_info['gateway_mac'],
                dst_mac=comp_ovsport.get_mac(),
                dst_port=comp_ovsport.get_ofport())

        if lvm.network_type == p_const.TYPE_VLAN:
            # TODO(vivek) remove the IPv6 related flows once SNAT is not
            # used for IPv6 DVR.
            br = self.phys_brs[lvm.physical_network]
        if lvm.network_type in constants.TUNNEL_NETWORK_TYPES:
            br = self.tun_br
        # TODO(vivek) remove the IPv6 related flows once SNAT is not
        # used for IPv6 DVR.
        if ip_version == 4:
            br.install_dvr_process_ipv4(
                vlan_tag=lvm.vlan, gateway_ip=subnet_info['gateway_ip'])
        else:
            br.install_dvr_process_ipv6(
                vlan_tag=lvm.vlan, gateway_mac=subnet_info['gateway_mac'])
        br.install_dvr_process(
            vlan_tag=lvm.vlan, vif_mac=port.vif_mac,
            dvr_mac_address=self.dvr_mac_address)

其中的 install\_dvr\_to\_src\_mac() 来自 br-int.py，具体实现为

    def install_dvr_to_src_mac(self, network_type,
                               vlan_tag, gateway_mac, dst_mac, dst_port):
        table_id = self._dvr_to_src_mac_table_id(network_type)
        self.add_flow(table=table_id,
                      priority=4,
                      dl_vlan=vlan_tag,
                      dl_dst=dst_mac,
                      actions="strip_vlan,mod_dl_src:%s,"
                      "output:%s" % (gateway_mac, dst_port))

用于剥除 VLAN Tag，并将目的 MAC 地址改为网关 MAC

而其中的 install\_dvr\_process\_ipv4() install\_dvr\_process\_ipv4() install\_dvr\_process() 函数均来自 br\_dvr\_process.py，具体实现如下

    def install_dvr_process_ipv4(self, vlan_tag, gateway_ip):
        # block ARP
        self.add_flow(table=self.dvr_process_table_id,
                      priority=3,
                      dl_vlan=vlan_tag,
                      proto='arp',
                      nw_dst=gateway_ip,
                      actions='drop')

    def install_dvr_process_ipv6(self, vlan_tag, gateway_mac):
        # block RA
        self.add_flow(table=self.dvr_process_table_id,
                      priority=3,
                      dl_vlan=vlan_tag,
                      proto='icmp6',
                      icmp_type=constants.ICMPV6_TYPE_RA,
                      dl_src=gateway_mac,
                      actions='drop')

    def install_dvr_process(self, vlan_tag, vif_mac, dvr_mac_address):
        self.add_flow(table=self.dvr_process_table_id,
                      priority=2,
                      dl_vlan=vlan_tag,
                      dl_dst=vif_mac,
                      actions="drop")
        self.add_flow(table=self.dvr_process_table_id,
                      priority=1,
                      dl_vlan=vlan_tag,
                      dl_src=vif_mac,
                      actions="mod_dl_src:%s,resubmit(,%s)" %
                      (dvr_mac_address, self.dvr_process_next_table_id))

分别用于丢弃 IPv4 的 ARP 请求、丢弃 IPv6 的 ICMP 请求、将数据包的源 MAC 地址改为 DVR MAC 地址

##DVR L3 Agent 源码实现

查看 neutron/agent/l3/agent.py 源码中的 \_create\_router() 函数，相关代码如下

    def _create_router(self, router_id, router):
        # TODO(Carl) We need to support a router that is both HA and DVR.  The
        # patch that enables it will replace these lines.  See bug #1365473.
        if router.get('distributed') and router.get('ha'):
            raise n_exc.DvrHaRouterNotSupported(router_id=router_id)

        args = []
        kwargs = {
            'router_id': router_id,
            'router': router,
            'use_ipv6': self.use_ipv6,
            'agent_conf': self.conf,
            'interface_driver': self.driver,
        }

        if router.get('distributed'):
            kwargs['agent'] = self
            kwargs['host'] = self.host
            if self.conf.agent_mode == l3_constants.L3_AGENT_MODE_DVR_SNAT:
                return dvr_router.DvrEdgeRouter(*args, **kwargs)
            else:
                return dvr_local_router.DvrLocalRouter(*args, **kwargs)

        if router.get('ha'):
            kwargs['state_change_callback'] = self.enqueue_state_change
            return ha_router.HaRouter(*args, **kwargs)

        return legacy_router.LegacyRouter(*args, **kwargs)

可见，\_create\_router() 函数根据 agent\_mode 的不同，创建不同的 DVR Router，如果是 SNAT DVR 节点，则创建 dvr\_router.DvrEdgeRouter，如果不是 SNAT DVR 节点，则创建 dvr\_local\_router.DvrLocalRouter；而 dvr\_router 是由 neutron/agent/l3/dvr_edge_router.py 导入，dvr\_local\_router 是由 neutron/agent/l3/dvr\_local\_router.py 导入

除了 dvr\_edge\_router.py 和 dvr\_local\_router.py 以外，neutron/agent/l3 文件夹下还有若干与 DVR L3 实现有关的源码，包括 dvr\_fip\_ns.py，dvr\_snat\_ns.py，dvr\_router\_base.py 和 dvr.py，这些源码文件间的继承层次关系如下图所示
，
<img src="http://www.deepurple.info/images/2015-8-17/dvr-l3.png" alt="image" width="50%"/>

dvr\_fip\_ns.py 和 dvr\_snat\_ns.py 主要实现了 FipNamespace 和 SnatNamespace 两个 Namespace，后续的 Floating IP 及 SNAT 相关操作均在相应的 Namespace 内部进行

dvr_local_router.py 主要实现了 DvrLocaRouter 类，DVR Local Router 运行在非 SNAT 的节点上

DvrLocalRouter 的主要处理逻辑位于 internal\_network\_added() 中，源码实现如下

    def internal_network_added(self, port):
        super(DvrLocalRouter, self).internal_network_added(port)

        # NOTE: The following function _set_subnet_arp_info
        # should be called to dynamically populate the arp
        # entries for the dvr services ports into the router
        # namespace. This does not have dependency on the
        # external_gateway port or the agent_mode.
        for subnet in port['subnets']:
            self._set_subnet_arp_info(subnet['id'])

        ex_gw_port = self.get_ex_gw_port()
        if not ex_gw_port:
            return

        sn_port = self.get_snat_port_for_internal_port(port)
        if not sn_port:
            return

        interface_name = self.get_internal_device_name(port['id'])
        self._snat_redirect_add(sn_port, port, interface_name)

在上面的源码中，首先调用 super() 方法，该 super() 方法最终沿着 DvrLocalRouter -> DvrRouterBase -> RouterInfo 的调用路径，调用了 router\_info.py 中的同名方法，该方法源码如下

    def internal_network_added(self, port):
        network_id = port['network_id']
        port_id = port['id']
        fixed_ips = port['fixed_ips']
        mac_address = port['mac_address']

        interface_name = self.get_internal_device_name(port_id)

        self._internal_network_added(self.ns_name,
                                     network_id,
                                     port_id,
                                     fixed_ips,
                                     mac_address,
                                     interface_name,
                                     INTERNAL_DEV_PREFIX)

该方法中调用了 \_internal\_network\_added()，从而将 Port Plug 到相应的 Driver（Bridge）中并设置好各种信息

调用 super() 之后，又调用了 \_snat\_redirect\_add()，设置到 SNAT 主机的路由，而 \_snat\_redirect\_add() 实际上是调用了 \_snat\_redirect\_modify()，在其中通过 sysctl 设置相应的路由

而在 DvrEdgeRouter 中也有相应的 internal\_network\_added() 函数，除了利用 super() 调用了 DvrLocalRouter 的上述处理逻辑外，还有如下的额外处理逻辑，将 Port Plug到相应的 Driver 里

    def internal_network_added(self, port):
        super(DvrEdgeRouter, self).internal_network_added(port)

        # TODO(gsagie) some of this checks are already implemented
        # in the base class, think how to avoid re-doing them
        if not self._is_this_snat_host():
            return

        sn_port = self.get_snat_port_for_internal_port(port)
        if not sn_port:
            return

        ns_name = dvr_snat_ns.SnatNamespace.get_snat_ns_name(self.router['id'])
        interface_name = self.get_snat_int_device_name(sn_port['id'])
        self._internal_network_added(
            ns_name,
            sn_port['network_id'],
            sn_port['id'],
            sn_port['fixed_ips'],
            sn_port['mac_address'],
            interface_name,
            dvr_snat_ns.SNAT_INT_DEV_PREFIX)

这样，Internal Network 和 External Network 就设置好了，剩下的就是 Floating IP 的相关处理逻辑

    def process_external(self, agent):
        ex_gw_port = self.get_ex_gw_port()
        if ex_gw_port:
            self.create_dvr_fip_interfaces(ex_gw_port)
        super(DvrLocalRouter, self).process_external(agent)

可以看到调用了 create\_dvr\_fip\_interfaces() 方法，该方法源码如下

    def create_dvr_fip_interfaces(self, ex_gw_port):
        floating_ips = self.get_floating_ips()
        fip_agent_port = self.get_floating_agent_gw_interface(
            ex_gw_port['network_id'])
        LOG.debug("FloatingIP agent gateway port received from the plugin: "
                  "%s", fip_agent_port)
        is_first = False
        if floating_ips:
            is_first = self.fip_ns.subscribe(self.router_id)
            if is_first and not fip_agent_port:
                LOG.debug("No FloatingIP agent gateway port possibly due to "
                          "late binding of the private port to the host, "
                          "requesting agent gateway port for 'network-id' :"
                          "%s", ex_gw_port['network_id'])
                fip_agent_port = self.agent.plugin_rpc.get_agent_gateway_port(
                    self.agent.context, ex_gw_port['network_id'])
                if not fip_agent_port:
                    LOG.error(_LE("No FloatingIP agent gateway port "
                                  "returned from server for 'network-id': "
                                  "%s"), ex_gw_port['network_id'])
            if is_first and fip_agent_port:
                if 'subnets' not in fip_agent_port:
                    LOG.error(_LE('Missing subnet/agent_gateway_port'))
                else:
                    self.fip_ns.create_gateway_port(fip_agent_port)

        if self.fip_ns.agent_gateway_port and floating_ips:
            if self.dist_fip_count == 0 or is_first:
                self.fip_ns.create_rtr_2_fip_link(self)

                # kicks the FW Agent to add rules for the IR namespace if
                # configured
                self.agent.process_router_add(self)

如果是第一个 Floating IP，则会调用 create\_gateway\_port()，从 Plugin 请求建立 Port，随后创建 Floating IP Namespace，并将 Gateway Port 添加到该 Namespace

调用 create\_dvr\_fip\_interfaces() 之后，又通过 super() 调用了 router\_info.py 中的 process\_external()，其中调用了 process\_snat\_dnat\_for\_fip()，利用 iptables 添加了相应的 SNAT 和 DNAT 规则

此外，在 external\_gateway\_added() 和 \_set\_subnet\_arp\_info() 等几个函数中，都调用了 \_update\_arp\_entry()，用于更新 DVR Local Router 的 ARP 表项

##DVR L3 Plugin 源码实现

查看 neutron/services/l3\_router/l3\_router\_plugin.py 源码，其中 L3RouterPlugin 类的初始化代码如下

    def __init__(self):
        self.setup_rpc()
        self.router_scheduler = importutils.import_object(
            cfg.CONF.router_scheduler_driver)
        self.start_periodic_l3_agent_status_check()
        super(L3RouterPlugin, self).__init__()
        if 'dvr' in self.supported_extension_aliases:
            l3_dvrscheduler_db.subscribe()
        l3_db.subscribe()

查看 neutron/db/l3\_dvrscheduler\_db.py 中 L3\_DVRsch\_db\_mixin() 中的 subscribe() 函数可知，这里的 l3\_dvrscheduler\_db.subscribe() 会订阅如下的几个 event

    def subscribe():
        registry.subscribe(
            _notify_l3_agent_new_port, resources.PORT, events.AFTER_UPDATE)
        registry.subscribe(
            _notify_l3_agent_new_port, resources.PORT, events.AFTER_CREATE)
        registry.subscribe(
            _notify_port_delete, resources.PORT, events.AFTER_DELETE)

继续查看 \_notify\_l3\_agent\_new\_port() 和 \_notify\_port\_delete() 两个函数的实现

\_notify\_l3\_agent\_new\_port() 中调用了 dvr\_vmarp\_table\_update() 和 dvr\_update\_router\_addvm()，dvr\_vmarp\_table\_table() 实现如下，由源码可知其根据不同的 action， 分别发送 add\_arp\_entry 和 del\_arp\_entry 消息给 L3

    def dvr_vmarp_table_update(self, context, port_dict, action):
        """Notify L3 agents of VM ARP table changes.

        When a VM goes up or down, look for one DVR router on the port's
        subnet, and send the VM's ARP details to all L3 agents hosting the
        router.
        """

        # Check this is a valid VM port
        if ("compute:" not in port_dict['device_owner'] or
            not port_dict['fixed_ips']):
            return
        ip_address = port_dict['fixed_ips'][0]['ip_address']
        subnet = port_dict['fixed_ips'][0]['subnet_id']
        filters = {'fixed_ips': {'subnet_id': [subnet]}}
        ports = self._core_plugin.get_ports(context, filters=filters)
        for port in ports:
            if port['device_owner'] == DEVICE_OWNER_DVR_INTERFACE:
                router_id = port['device_id']
                router_dict = self._get_router(context, router_id)
                if router_dict.extra_attributes.distributed:
                    arp_table = {'ip_address': ip_address,
                                 'mac_address': port_dict['mac_address'],
                                 'subnet_id': subnet}
                    if action == "add":
                        notify_action = self.l3_rpc_notifier.add_arp_entry
                    elif action == "del":
                        notify_action = self.l3_rpc_notifier.del_arp_entry
                    notify_action(context, router_id, arp_table)
                    return

dvr\_update\_router\_addvm() 实现如下，由源码可知其发送 routers\_updated 消息给 L3

    def dvr_update_router_addvm(self, context, port):
        ips = port['fixed_ips']
        for ip in ips:
            subnet = ip['subnet_id']
            filter_sub = {'fixed_ips': {'subnet_id': [subnet]},
                          'device_owner':
                          [n_const.DEVICE_OWNER_DVR_INTERFACE]}
            router_id = None
            ports = self._core_plugin.get_ports(context, filters=filter_sub)
            for port in ports:
                router_id = port['device_id']
                router_dict = self.get_router(context, router_id)
                if router_dict.get('distributed', False):
                    payload = {'subnet_id': subnet}
                    self.l3_rpc_notifier.routers_updated(
                        context, [router_id], None, payload)
                    break
            LOG.debug('DVR: dvr_update_router_addvm %s ', router_id)

\_notify\_port\_delete() 中也调用了 dvr\_vmarp\_table\_update()，此外还调用了 remove\_router\_from\_l3\_agent()，dvr\_vmarp\_table\_update() 的作用同前文所述，而 remove\_router\_from\_l3\_agent() 进行 Router 的删除操作

综上可知，\_notify\_l3\_agent\_new\_port() 和 \_notify\_port\_delete() 完成的主要工作为：

* \_notify\_l3\_agent\_new\_port()：如果一个 Port 有新增设备连接或 MAC 地址被修改，则发送 add\_arp\_entry 给 L3 Agent，修改对应的 ARP 流表规则；同时也会调用 routers\_updated 通知 L3，告知其 Router 有更新

* \_notify\_port\_delete的逻辑为：发送 del\_arp\_entry 给 L3 Agent，修改对应的 ARP 流表规则；如果某个 Port 删除后相关联的 Router 也可以删除了，那么会调用 remove\_router\_from\_l3\_agent 进行 Router 的删除

##参考链接

[Neutron/DVR](https://wiki.openstack.org/wiki/Neutron/DVR)

[Neutron/DVR L2 Agent](https://wiki.openstack.org/wiki/Neutron/DVR_L2_Agent)

[Neutron OVS DVR](http://specs.openstack.org/openstack/neutron-specs/specs/juno/neutron-ovs-dvr.html)

[OpenStack Neutron DVR L2 Agent初步解析（一）](http://blog.csdn.net/canxinghen/article/details/40978983)

[OpenStack Neutron DVR L2 Agent初步解析（二）](http://blog.csdn.net/canxinghen/article/details/41443127)

[初探OpenStack Neutron DVR](http://blog.csdn.net/matt_mao/article/details/39180135)

[DVR在Neutron中的代码实现](http://bingotree.cn/?p=706)

[DVR L2 Agent Blueprint](https://docs.google.com/document/d/1depasJSnGZPOnRLxEC_PYsVLcGVFXZLqP52RFTe21BE/edit)

[Enhanced L3 Agent Blueprint](https://docs.google.com/document/d/1jCmraZGirmXq5V1MtRqhjdZCbUfiwBhRkUjDXGt5QUQ/edit)

[OpenStack中的DVR Part1 - 东西向流量处置](http://www.myexception.cn/cloud/1702745.html)

[深入理解Neutron-OpenStack网络实现](http://yeasy.gitbooks.io/openstack_code_neutron/content/neutron/agent/l3_agentpy.html)
