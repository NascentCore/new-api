# 架构

HTTP Request
    │
    ▼
┌──────────┐   ┌──────────────┐   ┌────────────┐   ┌─────────┐   ┌──────────────────┐
│  Router  │ → │  Middleware  │ → │ Controller │ → │ Service │ → │ Model (GORM/DB)  │
└──────────┘   └──────────────┘   └────────────┘   └─────────┘   └──────────────────┘
                                        │
                                        ▼
                                  ┌───────────┐
                                  │   Relay   │ → 上游 AI Provider (HTTP)
                                  └───────────┘

## 关键数据模型关系图

User
 ├── Quota (钱包余额)
 ├── UsedQuota (已用额度)
 ├── StripeCustomer (Stripe 客户ID)
 │
 ├──< TopUp (充值记录)
 │    ├── TradeNo (订单号)
 │    ├── Amount (额度)
 │    ├── Money (金额)
 │    ├── PaymentMethod (stripe/epay/creem)
 │    └── Status (pending/success/expired)
 │
 ├──< UserSubscription (用户订阅)
 │    ├── PlanId → SubscriptionPlan
 │    ├── AmountTotal / AmountUsed
 │    ├── StartTime / EndTime
 │    ├── UpgradeGroup (升级用户组)
 │    └── QuotaResetPeriod (额度重置周期)
 │
 ├──< SubscriptionOrder (订阅订单)
 │    ├── TradeNo
 │    ├── PaymentMethod
 │    └── Status
 │
 └──< Log (使用日志)
      ├── LogTypeTopup (充值)
      ├── LogTypeConsume (消费)
      └── LogTypeRefund (退款)

SubscriptionPlan (订阅套餐)
 ├── PriceAmount / Currency
 ├── DurationUnit / DurationValue
 ├── TotalAmount (总额度)
 ├── StripePriceId (Stripe Price)
 ├── CreemProductId (Creem Product)
 └── QuotaResetPeriod / QuotaResetCustomSeconds

## 总结

这个项目的支付系统已经相当完善，核心设计模式是：

- 订单先行：支付前先创建 TopUp 或 SubscriptionOrder 记录（status=pending）
- 异步回调：支付完成后由支付提供商 Webhook 回调，后端验证签名后完成订单
- 幂等锁：LockOrder/UnlockOrder 防止重复回调
- 双源计费：钱包 + 订阅两个资金来源，用户可配置优先级
- 预扣结算：每次 AI 请求先预扣费，请求完成后按实际用量多退少补
