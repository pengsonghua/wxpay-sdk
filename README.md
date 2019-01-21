# wxpay-sdk
WeChat wxpay-sdk
基于微信官方sdk，版本3.0.9，修改了其中验签部分代码

验签方法没有签名方法参数，源代码在WXPay类中
~~~
public boolean isPayResultNotifySignatureValid(Map<String, String> reqData) throws Exception {
        String signTypeInData = reqData.get(WXPayConstants.FIELD_SIGN_TYPE);
        SignType signType;
        if (signTypeInData == null) {
            signType = SignType.MD5;
        }
        else {
            signTypeInData = signTypeInData.trim();
            if (signTypeInData.length() == 0) {
                signType = SignType.MD5;
            }
            else if (WXPayConstants.MD5.equals(signTypeInData)) {
                signType = SignType.MD5;
            }
            else if (WXPayConstants.HMACSHA256.equals(signTypeInData)) {
                signType = SignType.HMACSHA256;
            }
            else {
                throw new Exception(String.format("Unsupported sign_type: %s", signTypeInData));
            }
        }
        return WXPayUtil.isSignatureValid(reqData, this.config.getKey(), signType);
    }
~~~    
修改后代码
~~~
public boolean isPayResultNotifySignatureValid(Map<String, String> reqData, WXPayConstants.SignType signType) throws Exception {
        return WXPayUtil.isSignatureValid(reqData, this.config.getKey(), signType);
    }
~~~        
# 附
1.获取用户IP的方法
~~~
public class WXUtils {

    private WXUtils() {
    }

    public static String getIpAddr(HttpServletRequest request) {
        String ip = request.getHeader("X-Real-IP");
        if (!MtUtils.isNullOrEmpty(ip) && !"unknown".equalsIgnoreCase(ip))
            return ip;
        ip = request.getHeader("X-Forwarded-For");
        if (!MtUtils.isNullOrEmpty(ip) && !"unknown".equalsIgnoreCase(ip)) {
            int index = ip.indexOf(',');
            if (index != -1)
                return ip.substring(0, index);
            else
                return ip;
        }
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
            ip = request.getHeader("Proxy-Client-IP");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
            ip = request.getHeader("WL-Proxy-Client-IP");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
            ip = request.getHeader("HTTP_CLIENT_IP");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
            ip = request.getHeader("HTTP_X_FORWARDED_FOR");
        if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip))
            ip = request.getRemoteAddr();
        return ip;
    }
}
~~~
2.IWXPayDomain实现类
~~~
public class WXPayDomainImpl implements IWXPayDomain {

    private final int MIN_SWITCH_PRIMARY_MSEC = 3 * 60 * 1000;  //3 minutes
    private long switchToAlternateDomainTime = 0;
    private Map<String, DomainStatics> domainData = new HashMap<String, DomainStatics>();

    private WXPayDomainImpl() {
    }

    private static class WxpayDomainHolder {

        private static IWXPayDomain holder = new WXPayDomainImpl();
    }

    public static IWXPayDomain instance() {
        return WxpayDomainHolder.holder;
    }

    @Override
    public synchronized void report(String domain, long elapsedTimeMillis, Exception ex) {
        DomainStatics info = domainData.get(domain);
        if (info == null) {
            info = new DomainStatics(domain);
            domainData.put(domain, info);
        }
        if (ex == null) { //success
            if (info.succCount >= 2) {    //continue succ, clear error count
                info.connectTimeoutCount = info.dnsErrorCount = info.otherErrorCount = 0;
            } else {
                ++info.succCount;
            }
        } else if (ex instanceof ConnectTimeoutException) {
            info.succCount = info.dnsErrorCount = 0;
            ++info.connectTimeoutCount;
        } else if (ex instanceof UnknownHostException) {
            info.succCount = 0;
            ++info.dnsErrorCount;
        } else {
            info.succCount = 0;
            ++info.otherErrorCount;
        }
    }

    @Override
    public synchronized DomainInfo getDomain(WXPayConfig config) {
        DomainStatics primaryDomain = domainData.get(WXPayConstants.DOMAIN_API);
        if (primaryDomain == null || primaryDomain.isGood()) {
            return new DomainInfo(WXPayConstants.DOMAIN_API, true);
        }
        long now = System.currentTimeMillis();
        if (switchToAlternateDomainTime == 0) {   //first switch
            switchToAlternateDomainTime = now;
            return new DomainInfo(WXPayConstants.DOMAIN_API2, false);
        } else if (now - switchToAlternateDomainTime < MIN_SWITCH_PRIMARY_MSEC) {
            DomainStatics alternateDomain = domainData.get(WXPayConstants.DOMAIN_API2);
            if (alternateDomain == null || alternateDomain.isGood() || alternateDomain.badCount() < primaryDomain.badCount()) {
                return new DomainInfo(WXPayConstants.DOMAIN_API2, false);
            } else {
                return new DomainInfo(WXPayConstants.DOMAIN_API, true);
            }
        } else {  //force switch back
            switchToAlternateDomainTime = 0;
            primaryDomain.resetCount();
            DomainStatics alternateDomain = domainData.get(WXPayConstants.DOMAIN_API2);
            if (alternateDomain != null)
                alternateDomain.resetCount();
            return new DomainInfo(WXPayConstants.DOMAIN_API, true);
        }
    }

    static class DomainStatics {

        final String domain;
        int succCount = 0;
        int connectTimeoutCount = 0;
        int dnsErrorCount = 0;
        int otherErrorCount = 0;

        DomainStatics(String domain) {
            this.domain = domain;
        }

        void resetCount() {
            succCount = connectTimeoutCount = dnsErrorCount = otherErrorCount = 0;
        }

        boolean isGood() {
            return connectTimeoutCount <= 2 && dnsErrorCount <= 2;
        }

        int badCount() {
            return connectTimeoutCount + dnsErrorCount * 5 + otherErrorCount / 4;
        }
    }
}
~~~