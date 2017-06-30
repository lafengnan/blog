# TOTP基于时间的一次性口令算法

TOTP（Time-Based One-Time Password Algorithm)是目前各大网站以及移动APP的两步验证的重要技术支撑，其参考协议为[RFC6238](https://tools.ietf.org/html/rfc6238)。虽然不是严格的标准协议，但是对于了解TOTP的原理已经足够。



## RFC 6238

### 摘要

本文描述了定义在[RFC 4226](https://tools.ietf.org/html/rfc4226)中的名为"基于HMAC的一次性口令算法（HMAC-based One-Time Password, HOTP）"的扩展，以支持基于时间的进动因子。HOTP算法描述了一个基于时间的OTP算法，其进动因子是一个事件计数器。当前的工作则基于一个事件值作为进动因子。基于时间的OTP算法变体提供了短时效的OTP值，以期增强安全性。

推荐算法可以广泛应用于网络应用程序，从远程虚拟专用网（VPN）访问和Wi-Fi网络登录到面向事务的Web应用程序都可以使用。做着相信一个打通商业和开源实现之间的互操作性的通用共享算法可以促进“两步验证”在因特网上的普及应用。

### 本文状态

本文不是一篇因特网标准跟踪规范，仅用于信息目的。

本文是互联网工程任务小组（Internet Engineering Task Force, IETF)的工程成功。它代表IETF社区的共识。本文已经收到了公开的审查并且已经被互联网工程指导小组（Internet Engineering Steering Group, IESG)批准。并不是所有的经由IESG批准的文档都能成为互联网标准的；详情参考[Section 2 of RFC 5741](https://tools.ietf.org/html/rfc5741#section-2).

关于本文当前状态的信息，勘误，以及如何提供反馈请参考http://www.rfc-editor.org/info/rfc6238。

### 版权声明

```
 Copyright (c) 2011 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (http://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with respect
   to this document.  Code Components extracted from this document must
   include Simplified BSD License text as described in Section 4.e of
   the Trust Legal Provisions and are provided without warranty as
   described in the Simplified BSD License.
```

[TOC]

### 1. 介绍

#### 1.1 范畴

本文描述了定义在[RFC 4226](https://tools.ietf.org/html/rfc4226)中的名为"基于HMAC的一次性口令算法（HMAC-based One-Time Password, HOTP）"的扩展，以支持基于时间的进动因子。

#### 1.2 背景

正如[RFC4226](https://tools.ietf.org/html/rfc4226)中定义的，HOTP算法是基于*HMAC-SHA-1*算法（规范为[RFC2104](https://tools.ietf.org/html/rfc2104))的，并且使用一个递增的计算器值来表示HMAC计算中的消息。

总的来说，*HMAC-SHA-1*计算的输出被截断以期获得用户友好的值：

$$HOTP(K,C) = Truncate(HMAC-SHA-1(K,C)) $$

其中，Truncate表示该函数可以将一个*HMAC-SHA-1*值转换成一个HOTP值。$$K$$和$$C$$表示共享密钥和计数器值；详情参考[RFC4226](https://tools.ietf.org/html/rfc4226)。

TOTP是该算法（HOTP）基于时间的变体，其中用一个代表时间引用和事件步进的值$$T$$代替HOTP计算中的计数器$$C$$。

 TOTP实现**可以**使用基于SHA-256或者SHA-512的HMAC-SHA-256或者HMAC-SHA-512函数代替[RFC4226](https://tools.ietf.org/html/rfc4226)中描述的HOTP算法使用的HMAC-SHA-1函数。



### 2. 标记和术语

本文中的关键词“必须（MUST)”、"绝不(MUST NOT)"、 “必须的（REQUIRED)”、“应该（SHALL）“、”不应该（SHALL NOT）”、“应该（SHOULD）“、”不该（SHOULD NOT）“、”推荐（RECOMMENDED“、”可以（MAY）“，以及“可选的（OPTIONAL）”的解释在[RFC2119](https://tools.ietf.org/html/rfc2119)中描述。

### 3. 算法需求

这一节总结了设计TOTP算法需要考虑需求。

R1：证明者（比如，令牌（token）和软令牌（soft token））和验证者（鉴权或者验证服务器）**必须**知道或者有能力获得用于生成OTP的Unix时间（例如，从UTC时间1970年1月1日午夜以来流逝的秒数）。可以参考[UT](https://tools.ietf.org/html/rfc6238#ref-UT)获得通常所知的“Unix时间”的详细定义。证明者使用的时间精度会影响时钟同步必须完成的频次，详情参见[第6节](#6.再同步).

R2：证明者和验证者**必须**共享相同的密钥或者生成共享密钥的密钥转换知识。

R3：算法**必须**使用HOTP[RFC4226](https://tools.ietf.org/html/rfc4226)作为密钥基石。

R4：证明者和验证者**必须**使用相同的时间步进值$$X$$。

R5：每个证明者都必须**有一个唯一的密钥（secret key)。

R6：密钥**应该**随机产生或者使用密钥生成算法生成。

R7：密钥**可以**存储在一个防篡改的设备中，并且**应该**被保护而免遭未授权的访问和使用。

### 4. TOTP算法

这个HOTP算法变体描述了一个基于以时间因子作为计数器表述的一次性的口令值的计算方法。

#### 4.1 标记

- $$X$$表示以秒为单位的时间步进（默认值 $$X = 30$$秒），是一个系统参数
- $$T_0$$是开始启动时间步进计数的Unix时间（默认值是0，例如Unix epoch值），也是一个系统参数

#### 4.2 描述

基本上，我们将TOTP定义为$$TOTP = HOTP(K, T)$$，其中$$T$$是一个整数，用以表示初始计数器时间$$T_0$$与当前Unix时间之间的时间步进数（number of time step)。

进一步地讲，$$T = $$ (当前的Unix时间 - $$T_0) / X$$，计算中使用默认的地板除。举例来说，使$$T_0 = 0$$, 时间步进$$X = 30$$，那么如果当前的Unix时间是59秒，那么$$T = 1$$，如果当前的Unix时间是60秒则$$T = 2$$。

该算法的实现**必须**支持2038年以后大于32比特整数的时间值$$T$$。系统参数$$X$$和$$T_0$$的值在协商过程中预先建立，并且作为写生步骤的一部分在证明者和验证者之间交换。协商流程超出了本文的范畴，详情参考[RFC6030](https://tools.ietf.org/html/rfc6030)。

### 5. 安全性考量

#### 5.1 总则

该算法的安全性和强度依赖于底层基于使用SHA-1作为散列函数的HMAC[RFC2104](https://tools.ietf.org/html/rfc2104) HOTP基石。

安全性分析的详细结论在文档[RFC4226]()中，即，对于所有的实际使用目的来说，对不同的输入的动态截断都是均匀独立分布的字符串。

该分析表明，攻击HOTP函数的最佳方式是暴力攻击。

正如在算法需求一节中提示的，密钥**应该**随机的选择或者使用一个密码学意义上带有一个随机值作为种子的强伪随机数生成器生成。

密钥**应该**和HMAC的输出长度相同以适应互操作性。

我们**推荐**遵循[RFC4086](https://tools.ietf.org/html/rfc4086)中推荐的所有伪随机数和随机数生成器。用于生成谜语的伪随机数**应该**可以成功的通过[CN](https://tools.ietf.org/html/rfc6238#ref-CN)规定的随机数测试，或者通过类似的深受认可的测试。

所有的通信**应该**在一个安全信道上发生，比如安全套接字层/传输层安全（SSL/TLS)[RFC5246](https://tools.ietf.org/html/rfc5246)或者IPSec链接[RFC4301](https://tools.ietf.org/html/rfc4301)。

我们还**推荐**将密钥安全的存储在验证系统中，更具体的说，就是使用防篡改的硬件加密方法将密钥加密，只有当需要它们时才展示出来：例如，当需要验证一个OTP值时解密密钥，随后立即重新加密以限制密钥曝露在RAM中的时间。

密钥存储设备必须**在一个安全的区域，尽可能避免对验证系统和密钥数据库的直接攻击。实际操作中，限制只能由验证系统的程序和进程访问密钥的物理设备。



#### 5.2 验证与时间步进大小

在相同时间步进内生成的OTP是相同的。当一个验证系统收到一个OTP时，并不知道客户端产生OTP时的确切时间戳。验证系统典型情况下可能使用其收到OTP时的时间戳服务与OTP比较。由于网络延迟，OTP生成的时间与到达接收系统的时间之间的间隙（用$$T$$测量，即从$$T_0$$开始的时间步进数）可能很大。验证系统上的接受时间和实际的OTP生成时间可能并不处于生成OTP的那个相同的时间步进窗口中。当一个OTP在某个时间步进窗口的尾部产生时，接收时间很可能滑入下一个时间步进窗口中。验证系统**应该**为一个可接受的OTP传输延迟窗口验证设定一个典型的策略。验证系统应该不只用接收到的时间戳来比较OTP，而且还应该用传输延迟窗口内过去的时间戳比较。一个更大的可接受的延迟窗口会曝露一个更大的被攻击的时间窗。我们**推荐**最大只运行一个时间步进作为网络延迟。

时间步进的大小同时会影响安全性和可用性。一个更大的时间步进意味着对于一个可以被验证系统接受的OTP来说有一个更大的验证窗口。使用更大的时间步进有下列隐患：

首先，一个更大的时间步进值会曝露一个更大的攻击窗口。当产生一个OTP并将其告知一个第三方时，在该OTP被使用之前，这个第三方可以在时间窗口只能使用这个OTP。

我们**推荐**30秒作为默认的时间步进。这个默认的30秒是在安全性和可用性之间平衡的选择结果。

其次，必须在下一个时间步进窗口内生成下一个不同的OTP。从上一个提交之后，用户必须等待时钟到达下一个时间步进窗口，等待时间可能和事件步进的值并不完全相同，这取决于上一个OTP是何时生成的。举例来说，如果上一个OTP是在时间步进窗口的中间点产生的，那么等待时间就是步进窗口的一半。总的来说，一个更大的时间步进窗口对用户来说意味着在上一次成功的验证OTP之后获取下一个有效的OTP需要更长等待时间。一个过大的窗口（比如，10分钟）对于典型的互联网登录服务来说是不合适的。一个用户可能无法在10分钟内获得下一个OTP从而只能在10分钟内重新登录相同的网站。

需要注意的是，证明者可能会在一个给定的时间步进窗口内向验证者多次发送相同的OTP。验证者**绝不**可以在首次成功的验证OTP之后接受第二个试图验证的请求，以确保一个OTP的一次性特征。



### 6. 再同步

由于在客户端和验证服务器之间可能存在时钟漂移，我们**推荐**验证者可以设定一个证明者不被拒绝的可以“失去同步”的时间步进数限制。

这个限制可以对收到的OTP值的时间步进值的前后同时设置。如果时间步进是推荐的30秒，验证者设置了只接受向后2个步进，那么最大的时间漂移就是89秒。例如，计算出来的时间窗口内的29秒和向后的2个窗口的60秒。

这意味着验证者可以完成这样的验证，先验证当前时间，然后进一步验证向后的每一个步进（共3次验证）。根据成功的验证，验证服务器可以记录下侦测到的以时间步进数计算的时钟漂移。此后当收到一个新的OTP时，验证者可以用当前时间戳用记录下的时钟漂移的步进值修正后的值进行令牌验证。

当然，证明者没有向验证系统发送OTP的时间越长，证明者和验证者之间（隐藏）的累计时钟漂移就会越长，这非常重要。在这种情况下，如果漂移量超过了允许的阈值范围，上面描述的自动的重新同步可能不起作用。必须采用额外的鉴权来验证证明者权益并在证明者和验证者之间进行显式的时钟漂移同步。

### 7. 感谢

本文的作者非常感谢下列人员的共享和支持，使这篇文档成功更好的规范：Hannes Tschofenig, Jonathan Tuliani, David Dix,   Siddharth Bajaj, Stu Veath, Shuh Chang, Oanh Hoang, John Huang, and   Siddhartha Mohapatra.

### 8. 参考

#### 8.1 正式引用

  [RFC2104]  Krawczyk, H., Bellare, M., and R. Canetti, "HMAC: Keyed-Hashing for Message Authentication", RFC 2104, February 1997.

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate Requirement Levels", BCP 14, RFC 2119, March 1997.

   [RFC4086]  Eastlake 3rd, D., Schiller, J., and S. Crocker, "Randomness Recommendations for Security", BCP 106, RFC 4086, June 2005.

   [RFC4226]  M'Raihi, D., Bellare, M., Hoornaert, F., Naccache, D., and O. Ranen, "HOTP: An HMAC-Based One-Time Password Algorithm", RFC 4226, December 2005.

   [SHA2]     NIST, "FIPS PUB 180-3: Secure Hash Standard (SHS)", October 2008, <http://csrc.nist.gov/publications/fips/fips180-3/fips180-3_final.pdf>.

#### 8.2 非正式引用

   [CN]       Coron, J. and D. Naccache, "An Accurate Evaluation of Maurer's Universal Test", LNCS 1556, February 1999, <http://www.gemplus.com/smart/rd/publications/pdf/CN99maur.pdf>.

   [RFC4301]  Kent, S. and K. Seo, "Security Architecture for the Internet Protocol", RFC 4301, December 2005.

   [RFC5246]  Dierks, T. and E. Rescorla, "The Transport Layer Security (TLS) Protocol Version 1.2", RFC 5246, August 2008.

   [RFC6030]  Hoyer, P., Pei, M., and S. Machani, "Portable Symmetric Key Container (PSKC)", RFC 6030, October 2010.

   [UT]       Wikipedia, "Unix time", February 2011, <http://en.wikipedia.org/wiki/Unix_time>.

### 附录A TOTP算法：参考实现

``` java
/**
 Copyright (c) 2011 IETF Trust and the persons identified as
 authors of the code. All rights reserved.

 Redistribution and use in source and binary forms, with or without
 modification, is permitted pursuant to, and subject to the license
 terms contained in, the Simplified BSD License set forth in Section
 4.c of the IETF Trust's Legal Provisions Relating to IETF Documents
 (http://trustee.ietf.org/license-info).
 */

 import java.lang.reflect.UndeclaredThrowableException;
 import java.security.GeneralSecurityException;
 import java.text.DateFormat;
 import java.text.SimpleDateFormat;
 import java.util.Date;
 import javax.crypto.Mac;
 import javax.crypto.spec.SecretKeySpec;
 import java.math.BigInteger;
 import java.util.TimeZone;


 /**
  * This is an example implementation of the OATH
  * TOTP algorithm.
  * Visit www.openauthentication.org for more information.
  *
  * @author Johan Rydell, PortWise, Inc.
  */

 public class TOTP {

     private TOTP() {}

     /**
      * This method uses the JCE to provide the crypto algorithm.
      * HMAC computes a Hashed Message Authentication Code with the
      * crypto hash algorithm as a parameter.
      *
      * @param crypto: the crypto algorithm (HmacSHA1, HmacSHA256,
      *                             HmacSHA512)
      * @param keyBytes: the bytes to use for the HMAC key
      * @param text: the message or text to be authenticated
      */
    private static byte[] hmac_sha(String crypto, byte[] keyBytes,
             byte[] text){
         try {
             Mac hmac;
             hmac = Mac.getInstance(crypto);
             SecretKeySpec macKey =
                 new SecretKeySpec(keyBytes, "RAW");
             hmac.init(macKey);
             return hmac.doFinal(text);
         } catch (GeneralSecurityException gse) {
             throw new UndeclaredThrowableException(gse);
         }
     }


     /**
      * This method converts a HEX string to Byte[]
      *
      * @param hex: the HEX string
      *
      * @return: a byte array
      */

     private static byte[] hexStr2Bytes(String hex){
         // Adding one byte to get the right conversion
         // Values starting with "0" can be converted
         byte[] bArray = new BigInteger("10" + hex,16).toByteArray();

         // Copy all the REAL bytes, not the "first"
         byte[] ret = new byte[bArray.length - 1];
         for (int i = 0; i < ret.length; i++)
             ret[i] = bArray[i+1];
         return ret;
     }
   private static final int[] DIGITS_POWER
     // 0 1  2   3    4     5      6       7        8
     = {1,10,100,1000,10000,100000,1000000,10000000,100000000 };
    /**
      * This method generates a TOTP value for the given
      * set of parameters.
      *
      * @param key: the shared secret, HEX encoded
      * @param time: a value that reflects a time
      * @param returnDigits: number of digits to return
      *
      * @return: a numeric String in base 10 that includes
      *              {@link truncationDigits} digits
      */

     public static String generateTOTP(String key,
             String time,
             String returnDigits){
         return generateTOTP(key, time, returnDigits, "HmacSHA1");
     }


     /**
      * This method generates a TOTP value for the given
      * set of parameters.
      *
      * @param key: the shared secret, HEX encoded
      * @param time: a value that reflects a time
      * @param returnDigits: number of digits to return
      *
      * @return: a numeric String in base 10 that includes
      *              {@link truncationDigits} digits
      */

     public static String generateTOTP256(String key,
             String time,
             String returnDigits){
         return generateTOTP(key, time, returnDigits, "HmacSHA256");
     }
   /**
      * This method generates a TOTP value for the given
      * set of parameters.
      *
      * @param key: the shared secret, HEX encoded
      * @param time: a value that reflects a time
      * @param returnDigits: number of digits to return
      *
      * @return: a numeric String in base 10 that includes
      *              {@link truncationDigits} digits
      */

     public static String generateTOTP512(String key,
             String time,
             String returnDigits){
         return generateTOTP(key, time, returnDigits, "HmacSHA512");
     }


     /**
      * This method generates a TOTP value for the given
      * set of parameters.
      *
      * @param key: the shared secret, HEX encoded
      * @param time: a value that reflects a time
      * @param returnDigits: number of digits to return
      * @param crypto: the crypto function to use
      *
      * @return: a numeric String in base 10 that includes
      *              {@link truncationDigits} digits
      */

     public static String generateTOTP(String key,
             String time,
             String returnDigits,
             String crypto){
         int codeDigits = Integer.decode(returnDigits).intValue();
         String result = null;

         // Using the counter
         // First 8 bytes are for the movingFactor
         // Compliant with base RFC 4226 (HOTP)
         while (time.length() < 16 )
             time = "0" + time;

         // Get the HEX in a Byte[]
         byte[] msg = hexStr2Bytes(time);
         byte[] k = hexStr2Bytes(key);
        byte[] hash = hmac_sha(crypto, k, msg);

         // put selected bytes into result int
         int offset = hash[hash.length - 1] & 0xf;

         int binary =
             ((hash[offset] & 0x7f) << 24) |
             ((hash[offset + 1] & 0xff) << 16) |
             ((hash[offset + 2] & 0xff) << 8) |
             (hash[offset + 3] & 0xff);

         int otp = binary % DIGITS_POWER[codeDigits];

         result = Integer.toString(otp);
         while (result.length() < codeDigits) {
             result = "0" + result;
         }
         return result;
     }

     public static void main(String[] args) {
         // Seed for HMAC-SHA1 - 20 bytes
         String seed = "3132333435363738393031323334353637383930";
         // Seed for HMAC-SHA256 - 32 bytes
         String seed32 = "3132333435363738393031323334353637383930" +
         "313233343536373839303132";
         // Seed for HMAC-SHA512 - 64 bytes
         String seed64 = "3132333435363738393031323334353637383930" +
         "3132333435363738393031323334353637383930" +
         "3132333435363738393031323334353637383930" +
         "31323334";
         long T0 = 0;
         long X = 30;
         long testTime[] = {59L, 1111111109L, 1111111111L,
                 1234567890L, 2000000000L, 20000000000L};

         String steps = "0";
         DateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
         df.setTimeZone(TimeZone.getTimeZone("UTC"));
       try {
             System.out.println(
                     "+---------------+-----------------------+" +
             "------------------+--------+--------+");
             System.out.println(
                     "|  Time(sec)    |   Time (UTC format)   " +
             "| Value of T(Hex)  |  TOTP  | Mode   |");
             System.out.println(
                     "+---------------+-----------------------+" +
             "------------------+--------+--------+");

             for (int i=0; i<testTime.length; i++) {
                 long T = (testTime[i] - T0)/X;
                 steps = Long.toHexString(T).toUpperCase();
                 while (steps.length() < 16) steps = "0" + steps;
                 String fmtTime = String.format("%1$-11s", testTime[i]);
                 String utcTime = df.format(new Date(testTime[i]*1000));
                 System.out.print("|  " + fmtTime + "  |  " + utcTime +
                         "  | " + steps + " |");
                 System.out.println(generateTOTP(seed, steps, "8",
                 "HmacSHA1") + "| SHA1   |");
                 System.out.print("|  " + fmtTime + "  |  " + utcTime +
                         "  | " + steps + " |");
                 System.out.println(generateTOTP(seed32, steps, "8",
                 "HmacSHA256") + "| SHA256 |");
                 System.out.print("|  " + fmtTime + "  |  " + utcTime +
                         "  | " + steps + " |");
                 System.out.println(generateTOTP(seed64, steps, "8",
                 "HmacSHA512") + "| SHA512 |");

                 System.out.println(
                         "+---------------+-----------------------+" +
                 "------------------+--------+--------+");
             }
         }catch (final Exception e){
             System.out.println("Error : " + e);
         }
     }
 }
```



### 附录B 测试

本节提供了可以用于HOTP基于时间的变体算法的互操作性测试的测试值。

测试令牌使用ASCII字符串值“12345678901234567890”共享密钥。时间步进$$X = 30$$，Unix epoch作为初值来计算时间步进，其中$$T_0 = 0$$， TOTP算法会对给定模式和时间戳会显示下面的值。

```
  +-------------+--------------+------------------+----------+--------+
  |  Time (sec) |   UTC Time   | Value of T (hex) |   TOTP   |  Mode  |
  +-------------+--------------+------------------+----------+--------+
  |      59     |  1970-01-01  | 0000000000000001 | 94287082 |  SHA1  |
  |             |   00:00:59   |                  |          |        |
  |      59     |  1970-01-01  | 0000000000000001 | 46119246 | SHA256 |
  |             |   00:00:59   |                  |          |        |
  |      59     |  1970-01-01  | 0000000000000001 | 90693936 | SHA512 |
  |             |   00:00:59   |                  |          |        |
  |  1111111109 |  2005-03-18  | 00000000023523EC | 07081804 |  SHA1  |
  |             |   01:58:29   |                  |          |        |
  |  1111111109 |  2005-03-18  | 00000000023523EC | 68084774 | SHA256 |
  |             |   01:58:29   |                  |          |        |
  |  1111111109 |  2005-03-18  | 00000000023523EC | 25091201 | SHA512 |
  |             |   01:58:29   |                  |          |        |
  |  1111111111 |  2005-03-18  | 00000000023523ED | 14050471 |  SHA1  |
  |             |   01:58:31   |                  |          |        |
  |  1111111111 |  2005-03-18  | 00000000023523ED | 67062674 | SHA256 |
  |             |   01:58:31   |                  |          |        |
  |  1111111111 |  2005-03-18  | 00000000023523ED | 99943326 | SHA512 |
  |             |   01:58:31   |                  |          |        |
  |  1234567890 |  2009-02-13  | 000000000273EF07 | 89005924 |  SHA1  |
  |             |   23:31:30   |                  |          |        |
  |  1234567890 |  2009-02-13  | 000000000273EF07 | 91819424 | SHA256 |
  |             |   23:31:30   |                  |          |        |
  |  1234567890 |  2009-02-13  | 000000000273EF07 | 93441116 | SHA512 |
  |             |   23:31:30   |                  |          |        |
  |  2000000000 |  2033-05-18  | 0000000003F940AA | 69279037 |  SHA1  |
  |             |   03:33:20   |                  |          |        |
  |  2000000000 |  2033-05-18  | 0000000003F940AA | 90698825 | SHA256 |
  |             |   03:33:20   |                  |          |        |
  |  2000000000 |  2033-05-18  | 0000000003F940AA | 38618901 | SHA512 |
  |             |   03:33:20   |                  |          |        |
  | 20000000000 |  2603-10-11  | 0000000027BC86AA | 65353130 |  SHA1  |
  |             |   11:33:20   |                  |          |        |
  | 20000000000 |  2603-10-11  | 0000000027BC86AA | 77737706 | SHA256 |
  |             |   11:33:20   |                  |          |        |
  | 20000000000 |  2603-10-11  | 0000000027BC86AA | 47863826 | SHA512 |
  |             |   11:33:20   |                  |          |        |
  +-------------+--------------+------------------+----------+--------+
                             表 1： TOTP表
```

作者地址：

 David M'Raihi
   Verisign, Inc.
   685 E. Middlefield Road
   Mountain View, CA  94043
   USA

   EMail: davidietf@gmail.com


   Salah Machani
   Diversinet Corp.
   2225 Sheppard Avenue East, Suite 1801
   Toronto, Ontario  M2J 5C2
   Canada

   EMail: smachani@diversinet.com


   Mingliang Pei
   Symantec
   510 E. Middlefield Road
   Mountain View, CA  94043
   USA

   EMail: Mingliang_Pei@symantec.com


   Johan Rydell
   Portwise, Inc.
   275 Hawthorne Ave., Suite 119
   Palo Alto, CA  94301
   USA

   EMail: johanietf@gmail.com



## HOTP算法

由于TOTP算法是HOTP算法的变形，HOTP算法的详细规范为[RFC4226](https://tools.ietf.org/html/rfc4226)，这里简述一下该算法内容以方便整体串起来阅读。

### 符号表

算法表述：

1. 如果S是一个字符串，则| S |表示字符串长度。
2. 如果N是一个数值，则| N | 表示数值的绝对值。
3. 如果S是一个字符串，则S[i]表示S的第i位，起始位为0，S＝  S[0]S[1]S[2]…S[n-1]，n ＝ | S |
4. 函数StToNum表示将字符串S转化成其二进制数值，例如StToNum(110) = 6

符号表：

- **C **表示**8**字节的计数器，作为动进因子
- **K  **表示客户端和服务端的协商密钥，每个**HTOP**生成器都有一个唯一的密钥
- **T  **表示试错门限，在**T**次错误尝试之后，服务器将拒绝该客户端连接
- **S  **表示再同步参数，服务端在一个连续的计数范围内尝试验证接收到的验证码
- **Digit  **表示一个**HOTP**值的位数



### 描述

HOTP算法基于一个递增的计数器和一个仅对令牌与验证服务可见的静态对称密钥。为了创建HOTP值，我们会使用HMAC-SHA-1算法，其定义在规范[RFC 2104](https://tools.ietf.org/html/rfc2104) [BCK2]()中。

由于HMAC-SHA-1运算的输出是160比特，我们必须将该值截断成用户到可以很容易输入的值。

$$HOTP(K,C) = Truncate(HMAC{-}SHA{-}1(K, C))$$

其中，

- Truncate表示将一个HMAC-SHA-1值转换成一个HOTP值的函数。

密钥（$$K$$)，计数器（$$C$$)，和数据值首先从高端字节散列。

通过HOTP生成器产生的HOTP值以大端值处理。



### 产生HOTP值

操作由3个不同的步骤组成：

第一步：生成一个HMAC-SHA-1值，HS = HMAC-SHA-1(K,C) // HS是一个20字节的字符串

第二步：产生一个4字节字符串（动态截断）Sbits = DT(HS), // DT，在下面定义， 返回一个31比特字符串

第三步：计算一个HOTP值， Snum = StToNum(Sbits), //将S转换成一个 $$0… 2^{31}-1$$的数

返回 $$D = Snum \mod 10^{Digit}$$, // D是范围$$0...10^{Digit} - 1$$内的数

截断函数操作第二步和第三步。



#### DT定义

DT(String): 

​        OffsetBits = String[19]的低4位

​        Offset ＝ StToNum(OffsetBits) // 0 <= Offset <= 15

​        P = String[offset]… String[offset+3]

​        return P[1:31] // 返回P的后31位是为了避免符号位干扰

算法应该至少生成一个6位的数值码，也可以生成7位或者8位码，可以根据安全需要来调节。 下面的例子是Digit ＝ 6， 即生成6位HOTP码的示例：

```
   -------------------------------------------------------------
   | Byte Number                                               |
   -------------------------------------------------------------
   |00|01|02|03|04|05|06|07|08|09|10|11|12|13|14|15|16|17|18|19|
   -------------------------------------------------------------
   | Byte Value                                                |
   -------------------------------------------------------------
   |1f|86|98|69|0e|02|ca|16|61|85|50|ef|7f|19|da|8e|94|5b|55|5a|
   -------------------------------***********----------------++|
```

* 最后一个字节（第19字节）16进制值为0x5a
* 低4位是0xa（偏移量）
* 偏移量值是第10字节（0xa）
* 第10字节开始的4个字节：0x50ef7f19，是动态而金子码DBC1
* DBC1的MSB是0x50，因此DBC2 = DBC1 = 0x50ef7f19
* $$HOTP = DBC2 \mod 10^6 = 872921$$

我们将动态二进制码以31比特无符号大端整数处理，第一个字节标记为0x7f。将这个数值模$$1,000,000$$($$10^6$$)产生6位10进制数字HOTP值872921。