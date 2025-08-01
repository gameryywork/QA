# 分析
- 从两方面考虑：资源和通信
- 资源下载，一般采用https协议，常用框架xassets、addressables、TinaX.VFS、QFramework.ResKit等。
- 通信协议：通常采用https、tcp、udp、以及kcp类似的封装。
- 下面以bingo为例进行详细分析
一、网络架构整体特点
混合通信模式：实时 + 异步结合
游戏核心玩法（如 Bingo 对局）需要低延迟实时交互（如玩家喊 “Bingo”、道具使用反馈），采用长连接协议；而次要功能（如任务进度、奖励领取）则采用异步短连接，平衡性能与资源消耗。
强社交属性驱动的分布式架构
依赖 Facebook、Google 等社交平台的 API 实现好友互动，游戏服务器需与第三方社交服务实时通信，同时自身采用分布式集群支撑全球玩家（尤其欧美市场）的低延迟访问。
多区域部署与 CDN 加速
针对全球用户，采用多区域服务器集群（如北美、欧洲、亚太节点），结合 CDN 分发静态资源（如活动图片、音效），降低跨地域访问延迟。
二、核心网络模块设计
1. 通信协议分层
实时交互层（对局内）：
采用 WebSocket 或自定义 UDP 协议，原因是：
Bingo 对局中，玩家标记数字、使用道具、触发奖励等操作需毫秒级反馈，长连接可减少握手开销；
自定义 UDP 协议支持数据包优先级标记（如 “Bingo 确认” 请求优于 “聊天消息”），在弱网下优先保障核心玩法体验。
异步交互层（对局外）：
采用 HTTPS 协议处理非实时操作，如：
登录验证、好友列表拉取、任务进度同步；
商城购买、活动数据更新（每日奖励、限时赛事）。
优势是兼容性强（跨平台适配简单），且可利用 HTTP 缓存机制减少重复请求（如活动配置缓存到本地）。
数据格式：
实时数据（如对局状态）用二进制协议（如 Protobuf），减少传输体积，提升解析效率；
异步数据（如社交信息）用JSON，便于调试和与第三方社交平台对接。

综上，资源下载采用https；通信可以简化实现采用kcp，如果聊天压力效大，聊天服务使用单独服务器。

# 实现功能
- 仅以通信为例
- 基础通信能力：建立可靠的连接与传输
- 数据可靠性保障：解决 “传错、传丢、传重复”
- 离线与同步能力：适配 “时断时续” 的网络环境（一般可以单机游玩的游戏需要）
- 性能与体验优化：减少 “网络感”
- 监控与调试：快速定位问题
# 痛点
在休闲游戏的网络框架设计中，痛点往往围绕用户体验、稳定性、性能、安全性及运营需求展开，尤其需要适配休闲玩家 “碎片化、网络环境多变、对卡顿零容忍” 的特点。如下：
- 网络环境适配：应对复杂网络状态的鲁棒性
- 数据传输：效率与安全性的平衡
- 服务器压力与扩展性：应对用户量波动
- 用户体验：减少 “网络感知”
- 调试与监控：问题定位效率  
<br>
- 总结：休闲游戏网络框架的核心痛点，本质是在复杂网络环境下，如何以最低的性能损耗、最高的安全性，让用户感受不到网络的存在。设计时需结合场景优先保障：弱网、断网的容错性、数据一致性、用户操作的流畅性，同时预留服务器扩展和问题排查的灵活性，才能支撑游戏的长线运营。
<br>

# 代码部分

``` csharp
public delegate void OnRequestCallback(IResponseData data);

public interface IRequestData
{
    string Cmd {  get; }
    object Data { get; }
}

public interface IResponseData
{
    int Code { get; }
    string Error { get; }
    string Cmd { get; }
    object Data { get; }
}

public interface ISendConfig
{

}

/// <summary>
/// 基于不通框架进行的二次封装
/// </summary>
public interface IClient
{
    
    bool IsConnected { get; } 
    void Connect(string ip, int port);
    void Disconnect();
    void Reconnect();
    void Login();

    void Logout();

    /// <summary>
    /// 需要实现cache操作，未收到服务端返回，cache保留
    /// </summary>
    /// <param name="data"></param>
    /// <param name="callback"></param>
    /// <param name="config">实例特殊的配置需求</param>
    /// <returns></returns>
    int SendData(IRequestData data, OnRequestCallback callback = null, ISendConfig config = null);
    void OnReceiveData(byte[] data);

    void Heart();

    void Retry();

    void ClearCache();

    //休闲游戏创建房间可以采用逻辑上的房间，简化实现
}

/// <summary>
/// client管理
/// </summary>
public class NetworkManager
{
    public event Action OnConnect;
    public event Action OnDisconnect;
    public event Action<IResponseData> OnLogin;
    public event Action OnLogout;
    public event Action<IResponseData> OnBroadcast;
    //....

}
```
