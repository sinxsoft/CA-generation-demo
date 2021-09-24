# CA-generation-demo

双向认证：
如何使用Java访问双向认证的Https资源
本文的相关源码位于 https://github.com/dreamingodd/CA-generation-demo 

 

0.Nginx配置Https双向认证
首先配置Https双向认证的服务器资源。

可以参考：http://www.cnblogs.com/dreamingodd/p/7357029.html

完成之后如下效果：



 

1.导入cacerts进行访问
首先将服务器证书导入keystore cacerts，默认密码为changeit，如果需要修改密码就改一下。

keytool -import -alias ssl.demo.com -keystore cacerts -file C:\Development\deployment\ssl\ca-demo\server.crt
需要使用管理员权限到你使用的JDK security目录下执行（注意如果你有多个JDK的情况），效果如下：



然后使用Java访问：

复制代码
 1 package me.dreamingodd.ca;
 2 
 3 import org.apache.http.HttpEntity;
 4 import org.apache.http.client.methods.CloseableHttpResponse;
 5 import org.apache.http.client.methods.HttpGet;
 6 import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
 7 import org.apache.http.impl.client.CloseableHttpClient;
 8 import org.apache.http.impl.client.HttpClients;
 9 import org.apache.http.ssl.SSLContexts;
10 import org.apache.http.util.EntityUtils;
11 
12 import javax.net.ssl.SSLContext;
13 import java.io.File;
14 import java.io.FileInputStream;
15 import java.io.InputStream;
16 import java.security.KeyStore;
17 
18 
19 /**
20  * #1
21  * HTTPS 双向认证 - direct into cacerts
22  * @Author Ye_Wenda
23  * @Date 7/11/2017
24  */
25 public class HttpsKeyStoreDemo {
26     // 客户端证书路径，用了本地绝对路径，需要修改
27     private final static String PFX_PATH = "C:\\Development\\deployment\\ssl\\ca-demo\\client.p12";
28     private final static String PFX_PWD = "demo"; //客户端证书密码及密钥库密码
29 
30     public static String sslRequestGet(String url) throws Exception {
31         KeyStore keyStore = KeyStore.getInstance("PKCS12");
32         InputStream instream = new FileInputStream(new File(PFX_PATH));
33         try {
34             // 这里就指的是KeyStore库的密码
35             keyStore.load(instream, PFX_PWD.toCharArray());
36         } finally {
37             instream.close();
38         }
39 
40         SSLContext sslcontext = SSLContexts.custom().loadKeyMaterial(keyStore, PFX_PWD.toCharArray()).build();
41         SSLConnectionSocketFactory sslsf = new SSLConnectionSocketFactory(sslcontext
42                 , new String[] { "TLSv1" }  // supportedProtocols ,这里可以按需要设置
43                 , null  // supportedCipherSuites
44                 , SSLConnectionSocketFactory.getDefaultHostnameVerifier());
45 
46         CloseableHttpClient httpclient = HttpClients.custom().setSSLSocketFactory(sslsf).build();
47         try {
48             HttpGet httpget = new HttpGet(url);
49 //          httpost.addHeader("Connection", "keep-alive");// 设置一些heander等
50             CloseableHttpResponse response = httpclient.execute(httpget);
51             try {
52                 HttpEntity entity = response.getEntity();
53                 // 返回结果
54                 String jsonStr = EntityUtils.toString(response.getEntity(), "UTF-8");
55                 EntityUtils.consume(entity);
56                 return jsonStr;
57             } finally {
58                 response.close();
59             }
60         } finally {
61             httpclient.close();
62         }
63     }
64 
65     public static void main(String[] args) throws Exception {
66         System.out.println(sslRequestGet("https://ssl.demo.com/"));
67     }
68 
69 }
复制代码
运行结果如下：



 

2.生成truststore库文件进行访问-原生方式
如果服务器的JDK/JRE不能随便改动，我们还可以使用生成truststore库的方式来实现。

首先通过ca.crt生成自己的truststore，把ca.crt复制一份，重命名为ca.cer，复制到security目录下，执行

keytool -keystore demo.truststore -keypass demodemo -storepass demodemo -alias DemoCA -import -trustcacerts -file ca.cer
效果如下：



使用生成的demo.truststore和client.p12进行java访问：

复制代码
 1 package me.dreamingodd.ca;
 2 
 3 import javax.net.ssl.*;
 4 import java.io.*;
 5 import java.net.URL;
 6 import java.nio.charset.Charset;
 7 import java.security.KeyStore;
 8 
 9 
10 /**
11  * #2
12  * HTTPS 双向认证 - use truststore
13  * 原生方式
14  * @Author Ye_Wenda
15  * @Date 7/11/2017
16  */
17 public class HttpsTruststoreNativeDemo {
18     // 客户端证书路径，用了本地绝对路径，需要修改
19     private final static String CLIENT_CERT_FILE = "C:/Development/deployment/ssl/ca-demo/client.p12";
20     // 客户端证书密码
21     private final static String CLIENT_PWD = "demo";
22     // 信任库路径
23     private final static String TRUST_STRORE_FILE = "C:\\Development\\deployment\\ssl\\ca-demo\\demo.truststore";
24     // 信任库密码
25     private final static String TRUST_STORE_PWD = "demodemo";
26 
27 
28     private static String readResponseBody(InputStream inputStream) throws IOException {
29         try {
30             BufferedReader br = new BufferedReader(new InputStreamReader(inputStream, Charset.forName("UTF-8")));
31             StringBuffer sb = new StringBuffer();
32             String buff = null;
33             while((buff = br.readLine()) != null){
34                 sb.append(buff+"\n");
35             }
36             return sb.toString();
37         } finally {
38             inputStream.close();
39         }
40     }
41 
42     public static void httpsCall() throws Exception {
43         // 初始化密钥库
44         KeyManagerFactory keyManagerFactory = KeyManagerFactory
45                 .getInstance("SunX509");
46         KeyStore keyStore = getKeyStore(CLIENT_CERT_FILE, CLIENT_PWD, "PKCS12");
47         keyManagerFactory.init(keyStore, CLIENT_PWD.toCharArray());
48 
49         // 初始化信任库
50         TrustManagerFactory trustManagerFactory = TrustManagerFactory
51                 .getInstance("SunX509");
52         KeyStore trustkeyStore = getKeyStore(TRUST_STRORE_FILE, TRUST_STORE_PWD,"JKS");
53         trustManagerFactory.init(trustkeyStore);
54 
55         // 初始化SSL上下文
56         SSLContext ctx = SSLContext.getInstance("SSL");
57         ctx.init(keyManagerFactory.getKeyManagers(), trustManagerFactory
58                 .getTrustManagers(), null);
59         SSLSocketFactory sf = ctx.getSocketFactory();
60 
61         HttpsURLConnection.setDefaultSSLSocketFactory(sf);
62         String url = "https://ssl.demo.com";
63         URL urlObj = new URL(url);
64         HttpsURLConnection con = (HttpsURLConnection) urlObj.openConnection();
65         con.setRequestProperty("User-Agent", "Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36");
66         con.setRequestProperty("Accept-Language", "zh-CN;en-US,en;q=0.5");
67         con.setRequestMethod("GET");
68 
69         String response = readResponseBody(con.getInputStream());
70         System.out.println(response);
71     }
72 
73     /**
74      * 获得KeyStore
75      *
76      * @param keyStorePath
77      * @param password
78      * @return
79 
80      * @throws Exception
81      */
82     private static KeyStore getKeyStore(String keyStorePath, String password,String type)
83             throws Exception {
84         FileInputStream is = new FileInputStream(keyStorePath);
85         KeyStore ks = KeyStore.getInstance(type);
86         ks.load(is, password.toCharArray());
87         is.close();
88         return ks;
89     }
90 
91 
92     public static void main(String[] args) throws Exception {
93         httpsCall();
94     }
95 
96 }
复制代码
 

结果同1。

 

3.生成truststore库文件进行访问-Apache HTTP 组件方式
 

复制代码
package me.dreamingodd.ca;

import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.conn.ssl.SSLConnectionSocketFactory;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;

import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.TrustManagerFactory;
import java.io.*;
import java.net.URI;
import java.nio.charset.Charset;
import java.security.KeyStore;
import java.security.SecureRandom;

/**
 * #3
 * HTTPS 双向认证 - use truststore
 * Apache插件
 * @Author Ye_Wenda
 * @Date 7/11/2017
 */
public class HttpsTruststoreApacheContextDemo {
    // 客户端证书路径，用了本地绝对路径，需要修改
    private final static String CLIENT_CERT_FILE = "C:/Development/deployment/ssl/ca-demo/client.p12";
    // 客户端证书密码
    private final static String CLIENT_PWD = "demo";
    // 信任库路径
    private final static String TRUST_STRORE_FILE = "C:\\Development\\deployment\\ssl\\ca-demo\\demo.truststore";
    // 信任库密码
    private final static String TRUST_STORE_PWD = "demodemo";


    private static String readResponseBody(InputStream inputStream) throws IOException {
        try{
            BufferedReader br = new BufferedReader(new InputStreamReader(inputStream, Charset.forName("UTF-8")));
            StringBuffer sb = new StringBuffer();
            String buff = null;
            while((buff = br.readLine()) != null){
                sb.append(buff+"\n");
            }
            return sb.toString();
        }finally{
            inputStream.close();
        }
    }

    public static void httpsCall() throws Exception {
        // 初始化密钥库
        KeyManagerFactory keyManagerFactory = KeyManagerFactory
                .getInstance("SunX509");
        KeyStore keyStore = getKeyStore(CLIENT_CERT_FILE, CLIENT_PWD, "PKCS12");
        keyManagerFactory.init(keyStore, CLIENT_PWD.toCharArray());

        // 初始化信任库
        TrustManagerFactory trustManagerFactory = TrustManagerFactory
                .getInstance("SunX509");
        KeyStore trustkeyStore = getKeyStore(TRUST_STRORE_FILE, TRUST_STORE_PWD,"JKS");
        trustManagerFactory.init(trustkeyStore);

//        SSLContext sslContext = SSLContexts.custom().loadKeyMaterial(keyStore, "123456".toCharArray())
//            .loadTrustMaterial(new File(TRUST_STRORE_FILE),"012345".toCharArray()).setSecureRandom(new SecureRandom()).useProtocol("SSL").build();
        SSLContext sslContext = SSLContext.getInstance("SSL");
        sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), new SecureRandom());

        SSLConnectionSocketFactory sslConnectionSocketFactory = new SSLConnectionSocketFactory(sslContext,new String[]{"TLSv1", "TLSv2", "TLSv3"},null,
                SSLConnectionSocketFactory.getDefaultHostnameVerifier());

        CloseableHttpClient closeableHttpClient = HttpClients.custom().setSSLContext(sslContext).build();
        HttpGet getCall = new HttpGet();
        getCall.setURI(new URI("https://ssl.demo.com"));
        CloseableHttpResponse response = closeableHttpClient.execute(getCall);
        System.out.println(convertStreamToString(response.getEntity().getContent()));

    }

    public static String convertStreamToString(InputStream is) {
        /*
          * To convert the InputStream to String we use the BufferedReader.readLine()
          * method. We iterate until the BufferedReader return null which means
          * there's no more data to read. Each line will appended to a StringBuilder
          * and returned as String.
          */
        BufferedReader reader = new BufferedReader(new InputStreamReader(is));
        StringBuilder sb = new StringBuilder();

        String line = null;
        try {
            while ((line = reader.readLine()) != null) {
                sb.append(line + "\n");
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                is.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        return sb.toString();
    }

    /**
     * 获得KeyStore
     *
     * @param keyStorePath
     * @param password
     * @return
     * @throws Exception
     */
    private static KeyStore getKeyStore(String keyStorePath, String password,String type)
            throws Exception {
        FileInputStream is = new FileInputStream(keyStorePath);
        KeyStore ks = KeyStore.getInstance(type);
        ks.load(is, password.toCharArray());
        is.close();
        return ks;
    }



    public static void main(String[] args) throws Exception {
        httpsCall();
    }
}
复制代码
 

结果同2。
