本文使用nvidiaProfileInspector来设置

1.点击按钮 Show unknown settings from NVlDlA predefined profiles
2.在Extra项中找到OGL_DX_PRESENT_DEBUG，将其设置为0x000802A5




（可选）

1.然后在Other项中可以找到WKS MEMORY ALLOCATION POLICY ID，将其设置为0x00000002 WKS_MEMORY_ALLOCATION_POLICY_AGGRESSIVE_PRE_ALLOCATION（积极使用显存）

2.DLSS - Force DLSS preset 设置为Use latest preset version available in DLL
DLSS - Super Resolution Override 设置为On （强制所有支持dlss2+以上的游戏替换为最新dlss）
