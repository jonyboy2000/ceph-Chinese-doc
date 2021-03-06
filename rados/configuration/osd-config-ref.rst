==============
 OSD 配置参考
==============

.. index:: OSD; configuration

你可以通过配置文件调整 OSD ，但靠默认值和极少的配置 OSD 守护进程\
就能运行。最简 OSD 配置需设置 ``osd journal size`` 和 ``host`` ，\
其他几乎都能用默认值。

Ceph 的 OSD 守护进程用递增的数字作标识，按惯例以 ``0`` 开始，\
如下： ::

	osd.0
	osd.1
	osd.2

在配置文件里， ``[osd]`` 段下的配置适用于所有 OSD ；要添加针对特\
定 OSD 的选项（如 ``host`` ），把它放到那个 OSD 段下即可，如：

.. code-block:: ini

	[osd]
		osd journal size = 1024

	[osd.0]
		host = osd-host-a

	[osd.1]
		host = osd-host-b


.. index:: OSD; config settings

常规配置
========

下列选项可配置一 OSD 的唯一标识符、以及数据和日志的路径。 Ceph 部署脚本通常会自动生\
成 UUID 。我们\ **不建议**\ 更改数据和日志的默认路径，因为这样会增加后续的排障难度。

日志尺寸应该大于期望的驱动器速度和 ``filestore max sync interval`` 之乘积的两倍；\
最常见的方法是为日志驱动器（通常是 SSD ）分区并挂载好，这样 Ceph 就可以用整个分区做\
日志。


``osd uuid``

:描述: OSD 的全局唯一标识符（ UUID ）。
:类型: UUID
:默认值: The UUID.
:备注: ``osd uuid`` 适用于单个 OSD ， ``fsid`` 适用于整个集群。


``osd data``

:描述: OSD 数据存储位置，你得创建并把数据盘挂载到其下。我们不推荐更改默认值。
:类型: String
:默认值: ``/var/lib/ceph/osd/$cluster-$id``


``osd max write size``

:描述: 一次写入的最大尺寸，MB。
:类型: 32-bit Integer
:默认值: ``90``


``osd client message size cap``

:描述: 内存里允许的最大客户端数据消息。
:类型: 64-bit Integer Unsigned
:默认值: 默认为 500MB 。 ``500*1024L*1024L``


``osd class dir``

:描述: RADOS 类插件的路径。
:类型: String
:默认值: ``$libdir/rados-classes``


.. index:: OSD; file system

文件系统选项
============

Ceph 可自动创建并挂载所需的文件系统。


``osd mkfs options {fs-type}``

:描述: 为 OSD 新建 {fs-type} 类型的文件系统时使用的选项。
:类型: String
:xfs 默认值: ``-f -i 2048``
:其余文件系统默认值: {empty string}
:实例: ``osd mkfs options xfs = -f -d agcount=24``


``osd mount options {fs-type}``

:描述: 挂载 {fs-type} 类型的文件系统作为 OSD 数据目录时所用的选项。
:类型: String
:xfs 默认值: ``rw,noatime,inode64``
:其余文件系统默认值: ``rw, noatime``
:实例: ``osd mount options xfs = rw, noatime, inode64, logbufs=8``


.. index:: OSD; journal settings

日志选项
========

默认情况下， Ceph 觉得你会把 OSD 日志存储于下列路径： ::

	/var/lib/ceph/osd/$cluster-$id/journal

未做性能优化时， Ceph 会把日志存储在与 OSD 数据相同的硬盘上。追求高性能的 OSD 可用\
单独的硬盘存储日志数据，如固态硬盘能提供高性能日志。

``osd journal size`` 默认值是 0 ，所以你得在 ``ceph.conf`` 里设置。日志尺寸应该\
是 ``filestore max sync interval`` 与期望吞吐量的乘积再乘以 2 。 ::

	osd journal size = {2 * (expected throughput * filestore max sync interval)}

期望吞吐量应考虑期望的硬盘吞吐量（即持续数据传输速率）、和网络吞吐量，例如一个 \
7200 转硬盘的速度大致是 100MB/s 。硬盘和网络吞吐量中较小的（ ``min()`` ）一个是相\
对合理的吞吐量，有的用户则以 10GB 日志尺寸起步，例如： ::

	osd journal size = 10000


``osd journal``

:描述: OSD 日志路径，可以是一个文件或块设备（ SSD 的一个分区）的路径。如果是\
       文件，要先创建相应目录。我们建议用 ``osd data`` 以外的独立驱动器。

:类型: String
:默认值: ``/var/lib/ceph/osd/$cluster-$id/journal``


``osd journal size``

:描述: 日志尺寸（ MB ）。如果是 0 且日志文件是块设备，它会使用整个块设备。\
       从 v0.54 起，如果日志文件是块设备，这个选项会被忽略，且使用整个块设备。

:类型: 32-bit Integer
:默认值: ``5120``
:推荐值: 最少 1G ，应该是期望的驱动器速度和 ``filestore max sync interval`` \
         的乘积。


详情见\ `日志配置参考`_\ 。


监视器和 OSD 的交互
===================

OSD 周期性地相互检查心跳并报告给监视器。 Ceph 默认配置可满足多数情况，但是如果你的\
网络延时大，就得用较长间隔。关于心跳的讨论参见\ `监视器与 OSD 交互的配置`_\ 。



数据归置
========

详情见\ `存储池和归置组配置参考`_\ 。



.. index:: OSD; scrubbing

洗刷
====

除了为对象复制多个副本外， Ceph 还要洗刷归置组以确保数据完整\
性。这种洗刷类似对象存储层的 ``fsck`` ，对每个归置组， Ceph \
生成一个所有对象的目录，并比对每个主对象及其副本以确保没有对\
象丢失或错配。轻微洗刷（每天）检查对象尺寸和属性，深层洗刷（\
每周）会读出数据并用校验和方法确认数据完整性。

洗刷对维护数据完整性很重要，但会影响性能；你可以用下列选项来\
增加或减少洗刷操作。


``osd max scrubs``

:描述: 一 OSD 的最大并发洗刷操作数。
:类型: 32-bit Int
:默认值: ``1``


``osd scrub begin hour``

:描述: 被调度的洗刷操作在一天中可以运行的时间下限。
:类型: 0 到 24 之间的整数
:默认值: ``0``


``osd scrub end hour``

:描述: 被调度的洗刷操作在一天中可以运行的时间上限。本选项与 \
       ``osd scrub begin hour`` 一起定义了一个时间窗口，在此\
       期间可以进行洗刷操作。但是，在某个归置组的洗刷间隔超过 \
       ``osd scrub max interval`` 时，不管这个时间窗口是否合\
       适都会执行。
:类型: 0 到 24 之间的整数
:默认值: ``24``


``osd scrub during recovery``

:描述: 在恢复期间允许洗刷。有正在进行的恢复，且这里为 ``false``
       时，就会禁止调度新的洗刷（和深层洗刷）。已经在运行的洗\
       刷不受影响。对繁忙的集群来说，这样做可降低负载。
:类型: Boolean
:默认值: ``true``


``osd scrub thread timeout``

:描述: 洗刷线程最大死亡时值。
:类型: 32-bit Integer
:默认值: ``60``


``osd scrub finalize thread timeout``

:描述: 洗刷终结线程最大超时值。
:类型: 32-bit Integer
:默认值: ``60*10``


``osd scrub load threshold``

:描述: 最大负载，当前系统负载（ ``getloadavg()`` 所定义的）高\
       于此值时 Ceph 不会洗刷。默认 ``0.5`` 。

:类型: Float
:默认值: ``0.5``


``osd scrub min interval``

:描述: 集群负载低的时候，洗刷的最小间隔时间，秒。
:类型: Float
:默认值: 每天一次。 ``60*60*24``


``osd scrub max interval``

:描述: 不论集群负载如何，都要进行洗刷的时间间隔。
:类型: Float
:默认值: 每周一次。 ``7*60*60*24``


``osd scrub chunk min``

:描述: 单个操作可洗刷的最小对象块数。数据块在洗刷期间， Ceph \
       会阻塞别人向它写入。
:类型: 32-bit Integer
:默认值: 5


``osd scrub chunk max``

:描述: 单个操作可洗刷的最大对象块数。
:类型: 32-bit Integer
:默认值: 25


``osd scrub sleep``

:描述: 洗刷下一组数据块前等待的时间。增加此值会拖慢整个洗刷进\
       度，但对客户端操作没什么影响。

:类型: Float
:默认值: 0


``osd deep scrub interval``

:描述: 深层洗刷的间隔（完整地读所有数据）。
       ``osd scrub load threshold`` 不会影响此选项。

:类型: Float
:默认值: 每周一次。 ``60*60*24*7``


``osd scrub interval randomize ratio``

:描述: 在给某一归置组调度下一个洗刷作业时，给 \
       ``osd scrub min interval`` 增加个随机延时，这个延时是个\
       小于 ``osd scrub min interval`` \* \
       ``osd scrub interval randomized ratio`` 的随机值。所以\
       在实践中，这个默认设置会把洗刷操作随机地散布到允许的时\
       间窗口内，即 ``[1, 1.5]`` \*
       ``osd scrub min interval`` 。

:类型: Float
:默认值: ``0.5``


``osd deep scrub stride``

:描述: 深层洗刷时的读取尺寸。
:类型: 32-bit Integer
:默认值: 512 KB. ``524288``



.. index:: OSD; operations settings

操作数
======

操作数选项允许你设置用于服务的线程数，如果把
``osd op threads`` 设置为 ``0`` 就禁用了多线程。默认情况下，
Ceph 用 30 秒超时和 30 秒抗议时间来把握 2 个线程的运行情\
况。你可以调整客户端操作和恢复操作的优先程度来优化恢复期间\
的性能。


``osd op threads``

:描述: OSD 操作线程数， ``0`` 为禁用。增大数量可以增加请求处\
       理速度。

:类型: 32-bit Integer
:默认值: ``2``


``osd op queue``

:描述: 本选项用于配置 OSD 内各种操作（ ops ）的优先顺序。两\
       种队列都实现了严格子队列，出常规队列之前要先出子队列，\
       它与常规队列的实现机制不同。最初的 PrioritizedQueue
       (``prio``) 使用令牌桶系统，在令牌足够多时它会先处理\
       优先级高的队列；在令牌不够多时，则按优先级从低到高依\
       次处理。新的 WeightedPriorityQueue (``wpq``) 会根据\
       其优先级处理所有队列，以避免出现饥饿队列。在一部分
       OSD 负载高于其它的时， WPQ 应该有优势。此配置更改后需\
       重启。

:类型: String
:可选值: prio, wpq
:默认值: ``prio``


``osd op queue cut off``

:描述: 本选项用于配置把哪个优先级的操作放入严格队列、还是常\
       规队列。 ``low`` 这个选项会把所有复制操作以及优先级更\
       高的放入严格队列；而 ``high`` 选项只会把复制的确认反\
       馈操作以及优先级更高的发往严格队列。此选项设置为
       ``high`` 时，应该有助于缓解集群内某些 OSD 特别繁忙的\
       情形，尤其是配合 ``osd op queue`` 设置为 ``wpq`` 使用\
       时效果更佳。忙于处理副本流量的 OSD 们，如果没有这些配\
       置，它们的主副本（ primary client ）客户端往往比较空\
       闲。此配置更改后需重启。

:类型: String
:可选值: low, high
:默认值: ``low``


``osd client op priority``

:描述: 设置客户端操作优先级，它相对于
       ``osd recovery op priority`` 。

:类型: 32-bit Integer
:默认值: ``63``
:有效范围: 1-63


``osd recovery op priority``

:描述: 设置恢复优先级，其值相对于 ``osd client op priority`` 。
:类型: 32-bit Integer
:默认值: ``10``
:有效范围: 1-63


``osd scrub priority``

:描述: 设置洗刷操作的优先级。与 ``osd client op priority`` 有\
       关。

:类型: 32-bit Integer
:默认值: ``5``
:有效范围: 1-63


``osd snap trim priority``

:描述: 设置快照修建操作的优先级。与 ``osd client op priority``
       有关。

:类型: 32-bit Integer
:默认值: ``5``
:有效范围: 1-63


``osd op thread timeout``

:描述: OSD 线程超时秒数。
:类型: 32-bit Integer
:默认值: ``15``


``osd op complaint time``

:描述: 一个操作进行多久后开始抱怨。
:类型: Float
:默认值: ``30``


``osd disk threads``

:描述: 硬盘线程数，用于在后台执行磁盘密集型操作，像数据洗刷和\
       快照修复。

:类型: 32-bit Integer
:默认值: ``1``


``osd disk thread ioprio class``

:描述: 警告：只有 ``osd disk thread ioprio class`` 和 \
       ``osd disk thread ioprio priority`` 同时改为非默认值时\
       此配置才生效。 OSD 用 ioprio_set(2) 为磁盘线程设置 I/O
       调度分类（ ``class`` ），当前支持 ``idle`` 、 ``be`` 或
       ``rt`` 。其中， ``idle`` 类意味着磁盘线程的优先级低于其\
       它的 OSD 线程，适合需延缓洗刷操作的情形，如 OSD 正忙于\
       处理客户端操作。 ``be`` 是默认值，将设置与其它 OSD 线程\
       相同的优先级。 ``rt`` 意为磁盘线程的优先级将高于其它任何
       OSD 线程。注：只能与 Linux 内核的 CFQ 调度器配合使用。从
       Jewel 版起，洗刷操作已经不再由磁盘的 iothread 发起了，\
       请参考 OSD 优先级选项。

:类型: String
:默认值: 空字符串


``osd disk thread ioprio priority``

:描述: 警告：只有 ``osd disk thread ioprio class`` 和 \
       ``osd disk thread ioprio priority`` 同时改为非默认值时\
       此配置才生效。它通过 ioprio_set(2) 设置磁盘线程的 I/O \
       调度优先级（ ``priority`` ），优先级从最高的 0 到最低的
       7 。如果某主机上的所有 OSD 都在 ``idle`` 类中竞争 I/O \
       资源（即控制器拥塞了），那么你就可以用此选项把某 OSD 的\
       磁盘线程优先级调低为 7 ，其它优先级为 0 的 OSD 就获得优\
       先权了。注：只能与 Linux 内核的 CFQ 调度器配合使用。

:类型: 0 到 7 间的整数， -1 禁用此功能。
:默认值: ``-1``


``osd op history size``

:描述: 要跟踪的最大已完成操作数量。
:类型: 32-bit Unsigned Integer
:默认值: ``20``


``osd op history duration``

:描述: 要跟踪的最老已完成操作。
:类型: 32-bit Unsigned Integer
:默认值: ``600``


``osd op log threshold``

:描述: 一次显示多少操作日志。
:类型: 32-bit Integer
:默认值: ``5``



.. index:: OSD; backfilling

回填
====

当集群新增或移除 OSD 时，按照 CRUSH 算法应该重新均衡集群，它会\
把一些归置组移出或移入多个 OSD 以回到均衡状态。归置组和对象的\
迁移会导致集群运营性能显著降低，为维持运营性能， Ceph 用 \
backfilling 来执行此迁移，它可以使得 Ceph 的回填操作优先级低于\
用户读写请求。


``osd max backfills``

:描述: 单个 OSD 允许的最大回填操作数。
:类型: 64-bit Unsigned Integer
:默认值: ``1``


``osd backfill scan min``

:描述: 集群负载低时，回填操作时扫描间隔。
:类型: 32-bit Integer
:默认值: ``64``


``osd backfill scan max``

:描述: 回填操作时最大扫描间隔。
:类型: 32-bit Integer
:默认值: ``512``


``osd backfill retry interval``

:描述: 重试回填请求前等待秒数。
:类型: Double
:默认值: ``10.0``



.. index:: OSD; osdmap

OSD 运行图
==========

OSD 运行图反映集群中运行的 OSD 守护进程，斗转星移，图元增加。
Ceph 用一些选项来确保 OSD 运行图增大时仍运行良好。


``osd map dedup``

:描述: 允许删除 OSD 图里的重复项。
:类型: Boolean
:默认值: ``true``


``osd map cache size``

:描述: 缓存的 OSD 图个数。
:类型: 32-bit Integer
:默认值: ``500``


``osd map cache bl size``

:描述: OSD 进程中，驻留内存的 OSD 图缓存尺寸。
:类型: 32-bit Integer
:默认值: ``50``


``osd map cache bl inc size``

:描述: OSD 进程中，驻留内存的 OSD 图缓存增量尺寸。
:类型: 32-bit Integer
:默认值: ``100``


``osd map message max``

:描述: 每个  MOSDMap 图消息允许的最大条目数量。
:类型: 32-bit Integer
:默认值: ``100``



.. index:: OSD; recovery

恢复
====

当集群启动、或某 OSD 守护进程崩溃后重启时，此 OSD 开始与其它 \
OSD 们建立连接，这样才能正常工作。详情见\
`监控 OSD 和归置组`_\ 。

如果某 OSD 崩溃并重生，通常会落后于其他 OSD ，也就是没有同归置\
组内最新版本的对象。这时， OSD 守护进程进入恢复模式并检索最新\
数据副本，并更新运行图。根据 OSD 挂的时间长短， OSD 的对象和归\
置组可能落后得厉害，另外，如果挂的是一个失效域（如一个机柜），\
多个 OSD 会同时重生，这样恢复时间更长、更耗资源。

为保持运营性能， Ceph 进行恢复时会限制恢复请求数、线程数、对象\
块尺寸，这样在降级状态下也能保持良好的性能。


``osd recovery delay start``

:描述: 对等关系建立完毕后， Ceph 开始对象恢复前等待的时间\
       （秒）。

:类型: Float
:默认值: ``0``


``osd recovery max active``

:描述: 每个 OSD 一次可以处理的活跃恢复请求数量，增大此值能加速\
       恢复，但它们会增大集群负载。

:类型: 32-bit Integer
:默认值: ``15``


``osd recovery max chunk``

:描述: 一次推送的数据块的最大尺寸。
:类型: 64-bit Integer Unsigned
:默认值: ``8 << 20``


``osd recovery max single start``

:描述: 某个 OSD 恢复时，各 OSD 即将新开的最大恢复操作数量。
:类型: 64-bit Unsigned Integer
:默认值: ``1``


``osd recovery thread timeout``

:描述: 恢复线程最大死亡时值。
:类型: 32-bit Integer
:默认值: ``30``


``osd recover clone overlap``

:描述: 在数据恢复期间保留重叠副本。应该总是 ``true`` 。
:类型: Boolean
:默认值: ``true``


``osd recovery sleep``

:描述: 下一轮恢复或回填操作前睡眠的时间，单位为秒。增大此值会\
       减慢恢复操作，同时客户端操作受到的影响也小些了。

:类型: Float
:默认值: ``0``



.. Tiering

分级缓存选项
============


``osd agent max ops``

:描述: 在高速模式下，每个分级缓存代理同时执行刷回操作的最大数\
       量。

:类型: 32-bit Integer
:默认值: ``4``


``osd agent max low ops``

:描述: 在低速模式下，每个分级缓存代理同时执行刷回操作的最大数\
       量。
:类型: 32-bit Integer
:默认值: ``2``

关于在高速模式下，分级缓存代理何时刷回脏对象，见
`cache target dirty high ratio`_ 选项。


杂项
====


``osd snap trim thread timeout``

:描述: 快照修复线程最大死亡时值。
:类型: 32-bit Integer
:默认值: ``60*60*1``


``osd backlog thread timeout``

:描述: 积压线程最大死亡时值。
:类型: 32-bit Integer
:默认值: ``60*60*1``


``osd default notify timeout``

:描述: OSD 默认通告超时，秒。
:类型: 32-bit Integer Unsigned
:默认值: ``30``


``osd check for log corruption``

:描述: 根据日志文件查找数据损坏，会耗费大量计算时间。
:类型: Boolean
:默认值: ``false``


``osd remove thread timeout``

:描述: OSD 删除线程的最大死亡时值。
:类型: 32-bit Integer
:默认值: ``60*60``


``osd command thread timeout``

:描述: 命令线程最大超时值。
:类型: 32-bit Integer
:默认值: ``10*60``


``osd command max records``

:描述: 限制返回的丢失对象数量。
:类型: 32-bit Integer
:默认值: ``256``


``osd auto upgrade tmap``

:描述: 在旧对象上给 ``omap`` 使用 ``tmap`` 。
:类型: Boolean
:默认值: ``true``


``osd tmapput sets users tmap``

:描述: 只在调试时使用 ``tmap`` 。
:类型: Boolean
:默认值: ``false``


``osd preserve trimmed log``

:描述: 保留本该修剪掉的日志文件，但是会占用更多磁盘空间。
:类型: Boolean
:默认值: ``false``


``osd fast fail on connection refused``

:描述: 如果启用此选项，崩溃的 OSD 会即刻被已互联的 OSD 和监视\
       器们标记为 down （假设已崩溃 OSD 所在主机还活着）。禁用\
       此选项即可恢复原来的行为，代价是 I/O 操作中途若有 OSD \
       崩溃可能会导致较长时间的 I/O 停顿。
:类型: Boolean
:默认值: ``true``


.. _pool: ../../operations/pools
.. _监视器与 OSD 交互的配置: ../mon-osd-interaction
.. _监控 OSD 和归置组: ../../operations/monitoring-osd-pg#peering
.. _存储池和归置组配置参考: ../pool-pg-config-ref
.. _日志配置参考: ../journal-ref
.. _cache target dirty high ratio: ../../operations/pools#cache-target-dirty-high-ratio
