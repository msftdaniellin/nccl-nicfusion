NIC Fusion is an extension of Port Fusion in NCCL. It enables NCCL's Infiniband plugin to group together physical NICs (up to 4) together into logical NICs used by NCCL's core algorithms.

The reason to fuse any NICs together in the first place are as follows:
1. Some of NCCL's algorithms currently only work with 8 NICs, so on systems with 16 NICs, those algorithms could crash.
2. NCCL's core tuning code works best with 8 NICs - NCCL may tune itself to use too many channels when trying to drive 16 NICs.
3. Clusters are coming which will have, or appear to NCCL as, 16 or 32 NICs with 8 GPUs per-system.
4. This is a foundational feature which we can use to enhance future capabilities, such as NIC failover and dynamic load balancing.

The reason for NIC fusion as opposed to NCCL's existing port fusion is as follows:
1. Port fusion expects each port to present as a separate VF of the same PCI device.
2. Port fusion overwrote the physical properties of each NIC, presenting a false topology if the user wanted to dump it to a file.
3. Port fusion did not respect the user-provided topology when making a decision to merge NICs - it only used the pciPath the OS provided it with.

NIC Fusion should happen automatically in the following situations:
1. The network plugin in use must have defined the new network APIs to take advantage - makeVDevice, vDevices, getVProperties. If makeVDevice has not been implemented, NIC fusion will be skipped. This change implements NIC Fusion in the internal IB plugin, so that option will always be available for users of NCCL.

2. If topology code detects NICs meeting the specific merge level from the environment variable NCCL_NET_MERGE_LEVEL. The possible values for this are as follows:
	- LOC - Merge with self, aka disable merging NICs
	- PORT (default) - Merge with other NICs presenting as multiple functions of the same PCI device (same behavior as before)
	- PIX - Merge with other NICs directly under the same PCI switch
	- PXB - Merge with other NICs under the same PCI tree
	- PHB - Merge with other NICs under the same CPU host bridge
	- SYS - Merge with other NICs on the same system, potentially crossing CPU host bridges

3. If the user has forced NCCL to merge an arbitrary set of NICs using NCCL_NET_FORCE_MERGE. This is a semicolon delimited list of comma delimited NICs to merge, aka:
   - `mlx5_0,mlx5_1;mlx5_2,mlx5_3;mlx5_4,mlx5_5,mlx5_6;mlx5_7`
   Will merge mlx5_0,mlx5_1 into a single virtual NIC, mlx5_2,mlx5_3 into another, mlx5_4,mlx5_5,mlx5_6 into another, and finally mlx5_7 into its own. This example demonstrates that there is no requirement of symmetry with merged devices, although there's no guaruntee of good performance in all configurations.

NIC Fusion will automatically use the PCI paths in the provided topology, if specified, or simply use the values returned from system calls (realpath().)

NIC Fusion can always be disabled via NCCL_NET_MERGE_LEVEL=LOC. The environment variable NCCL_IB_MERGE_NICs=0 can also be set to force NCCL to not merge, although this will cause a ncclInvalidUsage error (this alone cannot make an application cleanly skip NIC fusion.) Therefore setting this in situations in which you don't want NICs to merge is good to enforce correctness, but NCCL_NET_MERGE_LEVEL=LOC is the preferred way.

For testing on GH200 with 2x CX-7s per node, run with NCCL_NET_MERGE_LEVEL=PHB
Note - NIC fusion will fail if two NICs are attempted to be merged with different link layers. In this case, disable the non-IB NICs using NCCL_IB_HCA=^eth_nic_0,eth_nic_1.
