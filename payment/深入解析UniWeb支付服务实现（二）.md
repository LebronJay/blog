# 深入剖析 UniWeb 支付服务：涵盖支付、退款、查询及回调处理

## 一、引言
在当今数字化支付的大环境下，支付服务的稳定性和功能完整性至关重要。`UniWebPaymentServiceImpl` 类实现了统一网页支付的多种功能，包括不同支付方式的处理、退款操作、支付状态查询以及支付回调处理等。本文将对该类的代码进行全面深入的分析，着重探讨支付回调之后的相关逻辑。

## 二、支付回调处理逻辑（`processNotification` 方法）

### 2.1 解析通知数据
当接收到支付通知时，首先会对通知数据进行解析，提取关键信息，如订单交易 ID、支付交易 ID 和交易状态。

```java
@Override
public void processNotification(String notification) {
    JSONObject responseData = JSONObject.parseObject(notification);
    String orderTransactionId = responseData.getString("clientTransactionId");
    String payTransactionId = responseData.getString("transactionId");
    String transactionStatus = responseData.getString("transactionStatus");
```

### 2.2 处理支付成功情况
如果支付状态为成功，则更新支付日志和相关业务订单的支付状态。

```java
 if (StringUtils.isNotBlank(transactionStatus) && "success".equalsIgnoreCase(transactionStatus)) {
        log.info("UniWebProcessNotification success orderTransactionId:{}", orderTransactionId);

        // 更新支付日志
        // ...

        // 更新业务订单支付状态
        // ...
    }
```

### 2.3 处理支付失败情况
如果交易状态不是成功，则记录支付失败信息。

```java
    else {
        log.info("UniWebProcessNotification failed");
        // 处理支付失败逻辑
    }
} else {
    log.error("UniWebProcessNotification failed");
}
```

## 三、其他重要功能

### 3.1 退款功能（`refund` 方法）
支持对已支付订单进行退款操作，包括日志记录、汇率换算和状态更新。

```java
@Override
@Transactional(rollbackFor = Exception.class)
public RefundResult refund(String orderId, String payTransactionId, String payType, Double refundAmount, String countryCode, String city) {
    RefundResult refundResult = new RefundResult();
    String transactionId = String.valueOf(uuidService.getUID());
    refundAmount = refundAmount * 100;

    // 保存退款日志

    // 发送退款请求
    LinkedHashMap<String, Object> requestData = new LinkedHashMap<>();
    requestData.put("amount", refundAmount.intValue());
    requestData.put("clientRefundId", orderId);
    requestData.put("transactionId", Long.valueOf(payTransactionId));
    JSONObject requestJson = new JSONObject(requestData);
    JSONObject responseData = getHttpResponse(refundUrl, requestJson.toString());

    String respCode = responseData.getString("code");
    if ("success".equals(respCode)) {
        JSONObject wechatPaymentPartnerDto = responseData.getJSONObject("data").getJSONObject("wechatPaymentPartnerDto");
        if (Objects.nonNull(wechatPaymentPartnerDto)) {
            String resultStatus = wechatPaymentPartnerDto.getString("resultStatus");
            if ("success".equals(resultStatus)) {
                String refundId = wechatPaymentPartnerDto.getString("refundId");
                refundResult.setSuccess(true);
                refundResult.setRefundTransactionId(refundId);

                // 修改日志退款状态
                org.bson.Document responseDoc = org.bson.Document.parse(String.valueOf(responseData));
                commonService.updateRefundResult(orderId, transactionId, refundId, responseDoc, true);
            } else {
                refundResult.setSuccess(false);
            }
        }
    } else {
        refundResult.setSuccess(false);
        refundResult.setFailReason(responseData.getString("message"));
    }
    return refundResult;
}
```

### 3.2 支付状态查询（`paymentInquiry` 和 `queryPayStatus` 方法）
提供了查询支付状态的功能，方便商家和用户了解订单的支付情况。

```java
@Override
public PayResultVO paymentInquiry(String orderId, String uniWebTransactionId) {
    PayResultVO paymentDTO = new PayResultVO();
    LinkedHashMap<String, Object> requestData = new LinkedHashMap<>();
    requestData.put("clientTransactionId", orderId);
    requestData.put("transactionId", Long.valueOf(uniWebTransactionId));
    JSONObject requestJson = new JSONObject(requestData);
    JSONObject responseData = getHttpResponse(queryUrl, requestJson.toString());

    String respCode = responseData.getString("code");
    if ("success".equals(respCode)) {
        JSONObject wechatPaymentPartnerDto = responseData.getJSONObject("data").getJSONObject("wechatPaymentPartnerDto");
        if (Objects.nonNull(wechatPaymentPartnerDto)) {
            paymentDTO.setPayTransactionId(wechatPaymentPartnerDto.getString("transactionId"));
            paymentDTO.setTransactionStatus(wechatPaymentPartnerDto.getString("transactionStatus"));
        }
    }
    return paymentDTO;
}

@Override
public boolean queryPayStatus(String orderId, String uniWebTransactionId) {
    LinkedHashMap<String, Object> requestData = new LinkedHashMap<>();
    requestData.put("clientTransactionId", orderId);
    requestData.put("transactionId", Long.valueOf(uniWebTransactionId));
    JSONObject requestJson = new JSONObject(requestData);
    JSONObject responseData = getHttpResponse(queryUrl, requestJson.toString());

    String respCode = responseData.getString("code");
    if ("success".equals(respCode)) {
        JSONObject wechatPaymentPartnerDto = responseData.getJSONObject("data").getJSONObject("wechatPaymentPartnerDto");
        if (Objects.nonNull(wechatPaymentPartnerDto)) {
            String transactionStatus = wechatPaymentPartnerDto.getString("transactionStatus");
            return "success".equals(transactionStatus);
        }
    }
    return false;
}
```

### 3.3 退款状态查询（`queryRefundStatus` 方法）
用于查询退款操作的状态，确保退款流程的透明度。

```java
@Override
public boolean queryRefundStatus(String orderId, String refundId) {
    LinkedHashMap<String, Object> requestData = new LinkedHashMap<>();
    requestData.put("clientRefundId", orderId);
    requestData.put("refundId", Long.valueOf(refundId));
    JSONObject requestJson = new JSONObject(requestData);
    JSONObject responseData = getHttpResponse(refundQueryUrl, requestJson.toString());

    String respCode = responseData.getString("code");
    if ("success".equals(respCode)) {
        JSONObject refundPartnerDto = responseData.getJSONObject("data").getJSONObject("refundPartnerDto");
        if (Objects.nonNull(refundPartnerDto)) {
            String resultStatus = refundPartnerDto.getString("resultStatus");
            String transactionId = refundPartnerDto.getString("transactionId");
            if ("success".equals(resultStatus) && orderId.equals(transactionId)) {
                return true;
            } else {
                return false;
            }
        }
    }
    return false;
}
```

## 四、总结
`UniWebPaymentServiceImpl` 类实现了一套完整的支付服务体系，包括多种支付方式的处理、退款操作、支付和退款状态查询以及支付回调处理。通过对支付回调的处理，确保了订单支付状态的及时更新和业务流程的顺利进行。同时，其他功能的实现也为商家和用户提供了便利，提高了支付服务的可靠性和透明度。在实际应用中，可根据具体需求对代码进行扩展和优化，以满足不同业务场景的要求。 