# new-api + 算力分销代理商运营系统 技术框图

基于功能清单与路线图文档，绘制完整技术框图。

## 技术框图（Mermaid 格式）

```mermaid
flowchart TB
    subgraph clients [客户端层]
        direction LR
        EndUser[终端用户]
        AgentUser[代理商]
        AdminUser[平台管理员]
    end

    subgraph gateway [网关与路由层]
        direction LR
        HostResolver[Host 解析]
        Router[路由]
        MW[鉴权/限流/跨域]
    end

    subgraph api [API 层]
        direction LR
        RelayAPI[中继 API]
        DashboardAPI[控制台 API]
        WebAPI[前端]
    end

    subgraph controller [控制器层]
        direction LR
        RelayCtrl[中继]
        UserCtrl[用户]
        AgentCtrl[代理商]
        AdminCtrl[管理员]
    end

    subgraph service [业务服务层]
        direction LR
        RelaySvc[中继/算力]
        AgentSvc[代理商/渠道/计费]
        UserSvc[用户/认证]
    end

    subgraph model [数据模型层]
        direction LR
        M1[用户/Token/渠道]
        M2[代理商/域名]
        M3[充值/提现/发票]
    end

    subgraph storage [存储层]
        direction LR
        DB[(DB)]
        Redis[(Redis)]
    end

    subgraph external [外部服务]
        direction LR
        AI[上游 AI]
        OAuth[OAuth]
        SMTP[邮件]
    end

    clients --> gateway --> api --> controller --> service --> model --> storage
    RelaySvc -.-> AI
    UserSvc -.-> OAuth
    AgentSvc -.-> SMTP
```

## 业务闭环流程图

```mermaid
flowchart LR
    subgraph biz [商业闭环]
        A1[代理商入驻]
        A2[用户充值]
        A3[用户模型消费]
        A4[代理商收益]
        A5[代理商提现]
    end
    A1 --> A2 --> A3 --> A4 --> A5
    A5 -.->|"持续运营"| A1
```

## 代理商差异化架构

```mermaid
flowchart TB
    subgraph request [请求进入]
        Req[HTTP 请求]
    end

    subgraph resolve [代理商识别]
        Host[Host 请求头]
        Host --> Lookup[域名 -> 代理商映射]
        Lookup --> AgentConfig[加载代理商配置]
    end

    subgraph config [代理商配置]
        Brand[品牌定制]
        Pricing[定价配置]
        OAuthConfig[OAuth 配置]
        SMTPConfig[SMTP 配置]
    end

    subgraph render [动态渲染]
        Logo[Logo]
        SiteName[网站名称]
        HomeHTML[首页内容]
        Footer[页脚]
        Menu[自定义菜单]
    end

    Req --> Host
    AgentConfig --> config
    config --> render
```

## 模块功能矩阵

| 模块 | 功能要点 | 对应路由/服务 |
|------|----------|---------------|
| 代理商基础架构 | 域名绑定、Host 识别、品牌定制 | middleware/Distribute, model/Agent |
| 用户注册与归属 | 自动绑定代理商、渠道码、跨域登录 | controller/User, service/User |
| 实名认证 | 申请、审核、加密存储 | controller/User, model/User |
| 账单管理 | 充值、发票、提现 | controller/Billing, service/Billing |
| 代理商用户管理 | 用户列表、搜索、详情、统计 | controller/Agent, service/Agent |
| 代理商渠道管理 | 渠道增删改查、统计 | controller/Agent, model/AgentChannel |
| 代理商数据看板 | 概览、消费明细、API 统计 | controller/Agent, service/Stats |
| 代理商系统设置 | 基础/定价/SEO/SMTP/OAuth/协议 | controller/Setting, model/AgentOption |
| 管理员-代理商 | 创建、编辑、额度、佣金 | controller/Admin, service/Admin |
| 管理员-用户 | 发票审核、提现审核 | controller/Admin |
