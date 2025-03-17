# 深入解析 UniWeb 支付服务实现：支持多支付类型的后端逻辑

在当今数字化支付的浪潮中，为用户提供多样化的支付方式是提升用户体验的关键。本文将深入剖析 `UniWebPaymentServiceImpl` 类的代码，探讨其如何实现多种支付方式，包括微信支付、微信 H5 支付和 PayNow 支付。

## 一、类概述
`UniWebPaymentServiceImpl` 类实现了 `UniWebPaymentService` 接口，主要负责处理统一网页支付相关的业务逻辑。它通过读取配置文件中的相关信息，如域名、店铺 ID、通知 URL 等，与外部支付服务进行交互。

```java
@Slf4j
@DubboService(protocol = "dubbo")
public class UniWebPaymentServiceImpl extends DecBaseService implements UniWebPaymentService {
    // 配置信息
    @Value("${uniWeb.domain}")
    private String domain;
    @Value("${uniWeb.storeId}")
    private String storeId;
    // 其他配置信息...
}
```

## 二、支持的支付方式

### 1. 微信支付（`wechatPay` 方法）
微信支付支持两种模式：微信小程序支付和微信 APP 支付。方法接收订单 ID、用户 OpenID、支付金额、支付问题类型和支付类型枚举作为参数。

```java
@Override
public Map<String, Object> wechatPay(String orderId, String openId, BigDecimal payAmount, String paymentIssueType, UniWebPayTypeEnum uniWebPayTypeEnum) {
    long amount = payAmount.multiply(new BigDecimal(100)).longValue();
    LinkedHashMap<String, Object> request = new LinkedHashMap<>();
    // 设置请求参数
    request.put("acquirerType", uniWebPayTypeEnum.getCode());
    request.put("amount", amount);
    // 其他参数...

    Map<String, Object> returnMap = new HashMap<>();
    PayResultVO resultVO = pay(request, paymentIssueType, Constants.PAY_TYPE.WECHAT_PAY, uniWebPayTypeEnum, null);

    JsonObject payResult = getPayResult(resultVO, uniWebPayTypeEnum);
    returnMap.put("success", true);
    returnMap.put("payResult", payResult.toString());
    returnMap.put("orderTransactionId", orderId);
    returnMap.put("payTransactionId", resultVO.getPayTransactionId());
    return returnMap;
}
```

### 2. 微信 H5 支付（`wechatH5Pay` 方法）
微信 H5 支付适用于在网页上进行支付的场景。除了基本的订单信息外，还需要提供用户的客户端 IP 和语言环境。

```java
@Override
public Map<String, Object> wechatH5Pay(String orderId, String openId, BigDecimal payAmount, String paymentIssueType, String payerClientIp, String locale) {
    UniWebPayTypeEnum uniWebPayTypeEnum = UniWebPayTypeEnum.WECHAT_H5_MERCHANT_PAY;
    long amount = payAmount.multiply(new BigDecimal(100)).longValue();
    LinkedHashMap<String, Object> request = new LinkedHashMap<>();
    // 设置请求参数
    request.put("acquirerType", uniWebPayTypeEnum.getCode());
    request.put("amount", amount);
    // 其他参数...

    Map<String, Object> returnMap = new HashMap<>();
    PayResultVO resultVO = pay(request, paymentIssueType, Constants.PAY_TYPE.WECHAT_PAY, uniWebPayTypeEnum, locale);

    returnMap.put("success", true);
    returnMap.put("qrCodeInBase64", resultVO.getQrCodeInBase64());
    returnMap.put("orderTransactionId", orderId);
    returnMap.put("payTransactionId", resultVO.getPayTransactionId());
    return returnMap;
}
```

### 3. PayNow 支付（`payNowPay` 方法）
PayNow 是新加坡的一种即时支付方式，通过二维码进行支付。该方法接收订单 ID、支付金额、支付问题类型和语言环境作为参数。

```java
@Override
public PayResultVO payNowPay(String orderId, BigDecimal payAmount, String paymentIssueType, String locale) {
    UniWebPayTypeEnum uniWebPayTypeEnum = UniWebPayTypeEnum.DBS_PAY_NOW_MERCHANT_PRESENTED_QR_PAY;
    long amount = payAmount.multiply(new BigDecimal(100)).longValue();
    LinkedHashMap<String, Object> request = new LinkedHashMap<>();
    // 设置请求参数
    request.put("acquirerType", uniWebPayTypeEnum.getCode());
    request.put("amount", amount);
    // 其他参数...
    return pay(request, paymentIssueType, Constants.PAY_TYPE.PAY_NOW, uniWebPayTypeEnum, locale);
}
```

## 三、核心支付逻辑（`pay` 方法）
`pay` 方法是处理支付请求的核心逻辑。它首先保存支付请求日志，然后向支付服务发送 HTTP 请求，并根据不同的支付类型解析响应结果。

```java
private PayResultVO pay(LinkedHashMap<String, Object> requestData, String paymentIssueType, String payType, UniWebPayTypeEnum uniWebPayTypeEnum, String locale) {
    // 保存日志
    String transactionId = String.valueOf(uuidService.getUID());
    String timeStamp = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyMMddHHmmss"));
    org.bson.Document requestDoc = new org.bson.Document();
    // 设置日志信息
    requestDoc.put("uniWebPayType", uniWebPayTypeEnum.getCode());
    requestDoc.put("source", "uniweb");
    // 其他日志信息...
    commonService.savePayRequest(orderId, transactionId, payType, paymentIssueType,
            requestDoc, DecDateTime.nowLocalDateTime(countryCode, city), PaymentServiceProviderEnum.UNIWEB.getCode());

    PayResultVO paymentDTO = new PayResultVO();

    JSONObject requestJson = new JSONObject(requestData);
    JSONObject responseData = getHttpResponse(payUrl, requestJson.toString());
    String respCode = responseData.getString("code");
    if ("success".equals(respCode)) {
        org.bson.Document payResultDoc = null;
        String payTransactionId = null;
        switch (uniWebPayTypeEnum) {
            case MINI_PROGRAM_WECHAT_MERCHANT_PAY:
                // 处理微信小程序支付响应
                JSONObject miniProgramWechatPaymentPartnerDto = responseData.getJSONObject("data").getJSONObject("miniProgramWechatPaymentPartnerDto");
                if (Objects.nonNull(miniProgramWechatPaymentPartnerDto)) {
                    paymentDTO.setPayTransactionId(miniProgramWechatPaymentPartnerDto.getString("transactionId"));
                    paymentDTO.setPrepayId(miniProgramWechatPaymentPartnerDto.getString("prePayId"));
                    // 其他设置...
                }
                break;
            case WECHAT_IN_APP_MERCHANT_PAY:
                // 处理微信 APP 支付响应
                break;
            case WECHAT_H5_MERCHANT_PAY:
                // 处理微信 H5 支付响应
                break;
            case DBS_PAY_NOW_MERCHANT_PRESENTED_QR_PAY:
                // 处理 PayNow 支付响应
                break;
            default:
                // 处理默认情况
                break;
        }
    }
    return paymentDTO;
}
```

## 四、总结
`UniWebPaymentServiceImpl` 类通过统一的接口实现了多种支付方式的处理，包括微信支付、微信 H5 支付和 PayNow 支付。它通过配置文件管理支付相关信息，确保代码的可维护性和扩展性。在处理支付请求时，会保存支付日志，并根据不同的支付类型解析响应结果，为后续的业务处理提供了基础。

通过这种方式，开发者可以方便地集成多种支付方式，为用户提供更加便捷的支付体验。同时，代码的模块化设计也使得后续的功能扩展和维护变得更加容易。