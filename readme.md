# **eos-crypto-java 使用指南**

源码：https://github.com/yanjunli/eos-crypto-java

eos-crypto-java 目前可以支持 基于 ECC+AES 的加解密方式。 

在本压缩包中，包含基于jdk1.5 打好的jar 包。


## 要求

jdk 1.5+

## 基于 EOS 公钥加密，私钥解密示例

```java
        String privateKey =  "5KTZYCDdcfNrmEpcf97SJBCtToZjYHjHm8tqTWvzUbsUJgkxcfk";
        EosPrivateKey eosPrivateKey = new EosPrivateKey(privateKey);
        EosPublicKey  eosPublicKey = eosPrivateKey.getPublicKey();
        // 转换成 EC privatekey
        ECPrivateKey ecPrivateKey = eosPrivateKey.getECPrivateKey();
        ECPublicKey ecPublicKey = eosPublicKey.getECPublicKey();

        byte[] plaindata = "{\"age\": 1,\"12345\":\"24qqwazzxdtttdxkaskjewuizckczxnlsdosasda4!!!@#$$%^&&*(()(^#\"}".getBytes("utf8");

        System.out.println("加密原文：" + new String(plaindata));

        byte[] encryptdata = ECCUtil.publicEncrypt(plaindata,ecPublicKey);

        System.out.println("加密后密文：" + HexUtils.toHex(encryptdata));

        plaindata = ECCUtil.privateDecrypt(encryptdata,ecPrivateKey);

        System.out.println("解密后原文: "+ new String(plaindata));
```

## 基于ECC+AES 双向验证 加解密示例

```java
/**
*
* sender  发起方密钥对
*
* EOS8g1u3ktAGHs4QsVp9aeaWNebFLtprQHwpaSjegx6iEuoTNhjXU
* 5KTZYCDdcfNrmEpcf97SJBCtToZjYHjHm8tqTWvzUbsUJgkxcfk
*
* receiver 接收方一密钥对
*
* EOS7ez2gagfoXw9XdW3kRx3EsCoWvupGR6u6ZJhFPEe9Q12V8JgUL
* 5JUrqxYcssR9LLVtWDeQcc9HCX4FEqBG7d9GW6t7mvmB1rUuZr9
*
* receiver 接收方二 密钥对
* EOS5WMHqw6jDDBPBm7JXTHemAwtSo2tp93pRysJMRhiT1zUYb24vL
* 5HrcVeuHHNwHsivrMoJ9XvU6EM7Q2wQ2ECiy8GeoiuamhNiSuZq
*/

// 1.  调用钱包获取 发送方私钥
  String senderPrivateKey =  "5KTZYCDdcfNrmEpcf97SJBCtToZjYHjHm8tqTWvzUbsUJgkxcfk";
  EosPrivateKey senderECPrivateKey = new EosPrivateKey(senderPrivateKey);
//        EosPublicKey senderECPublicKey = new EosPublicKey(senderPublicKey);
        // 2.  根据私钥 生成公钥。 或者直接根据公钥 调用钱包获取私钥。 都可以。
  EosPublicKey senderECPublicKey = senderECPrivateKey.getPublicKey();

  String senderPublicKey = senderECPublicKey.toString();
  /**
   * 调用钱包获取 接收方私钥   获取公私钥方式 根据需求确定。
   *  1. 可以根据公钥，从钱包里获取私钥
   *  2. 也可以直接从钱包里取出私钥，反向生成公钥
   *  
   *  实际业务场景，发起方只会有接收方公钥，并没有接收方私钥. 
   *  此时 可以通过 new EosPublicKey(receiverPublicKey) 方式 生成EosPublicKey 对象。
   */
  String receiverPrivateKey = "5JUrqxYcssR9LLVtWDeQcc9HCX4FEqBG7d9GW6t7mvmB1rUuZr9";
  EosPrivateKey receiverECPrivateKey = new EosPrivateKey(receiverPrivateKey);
  EosPublicKey receiverECPublicKey = receiverECPrivateKey.getPublicKey();
  String receiverPublicKey = receiverECPublicKey.toString();
  //        String receiverPublicKey =  "EOS7ez2gagfoXw9XdW3kRx3EsCoWvupGR6u6ZJhFPEe9Q12V8JgUL";


  /**
   * 使用 发送者方私钥 和接收方公钥，生成 aes key, 对数据进行加密
   * nonce  为初始化向量，可以使用固定值，
   *                      也可以使用随机值，并使用私有协议。根据业务需求选择。
   */
  byte[] nonce = new byte[16];
  MTRandom random=new MTRandom();
  random.nextBytes(nonce);

  // 待加密 数据
  byte[] params = "{\"test1\": 1,\"test2\":\"24qqwazzxdtttdxkaskjewuizckczxnlsdosasda4!!!@#$$%^&&*((){}(^#\"}".getBytes("utf8");

  System.out.println("原始加密数据： " + new String(params,"utf8"));

  byte[] encrypted = new byte[0];
  try {
      encrypted = CryptUtil.encrypt(senderECPrivateKey,receiverECPublicKey,nonce,params);
  } catch (InvalidCipherTextException e) {
      e.printStackTrace();
      System.out.println("  do something!!!!");
  }

  System.out.println("加密后数据： " + new String(encrypted,"utf8"));
  try {
      byte[] plainText = CryptUtil.decrypt(receiverECPrivateKey,senderECPublicKey,nonce,encrypted);
      // 解密后数据
      System.out.println("解密后数据 :  "+new String(plainText, "utf8"));
  } catch (InvalidCipherTextException e) {
      e.printStackTrace();
      System.out.println("  do something!!!!");
  }
```

## 基于数字信封的 加解密示例

```java

/**
         *
         * sender
         *
         * EOS8g1u3ktAGHs4QsVp9aeaWNebFLtprQHwpaSjegx6iEuoTNhjXU
         * 5KTZYCDdcfNrmEpcf97SJBCtToZjYHjHm8tqTWvzUbsUJgkxcfk
         *
         * receiver 平台公私钥对
         *
         * EOS7ez2gagfoXw9XdW3kRx3EsCoWvupGR6u6ZJhFPEe9Q12V8JgUL
         * 5JUrqxYcssR9LLVtWDeQcc9HCX4FEqBG7d9GW6t7mvmB1rUuZr9
         *
         * receiver 省侧公私钥对
         * EOS5WMHqw6jDDBPBm7JXTHemAwtSo2tp93pRysJMRhiT1zUYb24vL
         * 5HrcVeuHHNwHsivrMoJ9XvU6EM7Q2wQ2ECiy8GeoiuamhNiSuZq
         */

        // 1.  调用钱包获取 发送方私钥
        String senderPrivateKey =  "5KTZYCDdcfNrmEpcf97SJBCtToZjYHjHm8tqTWvzUbsUJgkxcfk";
        EosPrivateKey senderECPrivateKey = new EosPrivateKey(senderPrivateKey);
//        EosPublicKey senderECPublicKey = new EosPublicKey(senderPublicKey);
        // 2.  根据私钥 生成公钥。 或者直接根据公钥 调用钱包获取私钥。 都可以，看具体业务需求
        EosPublicKey senderECPublicKey = senderECPrivateKey.getPublicKey();

        String senderPublicKey = senderECPublicKey.toString();
        /**
         * 调用钱包获取 接收方私钥   获取公私钥方式 根据业务需求确定。
         *  1. 可以根据公钥，从钱包里获取私钥
         *  2. 也可以直接从钱包里取出私钥，反向生成公钥
         */
        String receiverPrivateKey = "5JUrqxYcssR9LLVtWDeQcc9HCX4FEqBG7d9GW6t7mvmB1rUuZr9";
        EosPrivateKey receiverECPrivateKey = new EosPrivateKey(receiverPrivateKey);

        EosPublicKey receiverECPublicKey = receiverECPrivateKey.getPublicKey();

        /**
         * 生成对称密钥
         */
        byte[] nonce = new byte[16];
        MTRandom random=new MTRandom();
        random.nextBytes(nonce);

        // 待加密 数据
        byte[] params = "{\"age\": 1,\"汉字\":\"为初始化向量，可以使用固定值，\"，\"12345\":\"24qqwazzxdtttdxkaskjewuizckczxnlsdosasda4!!!@#$$%^&&*(()(^#\"}".getBytes("utf8");

        System.out.println("加密前原始数据： " + new String(params,"utf8"));

        // 发起方使用对称密钥，对原始数据进行加密
        byte[] encryptedData = null;
        try {
            encryptedData = CryptUtil.aesEncryptWithNOIV(nonce,params);
        } catch (InvalidCipherTextException e) {
            e.printStackTrace();
            System.out.println("  do something!!!!");
        }

        System.out.println("加密后数据： " + HexUtils.toHex(encryptedData));


        System.out.println("加密前对称密钥： " + HexUtils.toHex(nonce));

        // 发起方使用 接收方公钥，对对称密钥进行加密
        byte[] encryptedKey = null;
        try {
            encryptedKey = ECCUtil.publicEncrypt(nonce,receiverECPublicKey.getECPublicKey());
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("  do something!!!!");
        }

        System.out.println("加密后对称密钥： " + HexUtils.toHex(encryptedKey));

        // 将对称密钥加密后的数据，密文组装后，进行网络传输。
        // 组装 demo
        /**
         *    4 byte                       |      encryptedKey                 |       4 byte              | encryptedData
         *    对称密钥加密后的数据长度      |      ECC 加密后的对称秘钥           |       密文数据长度         | AES 加密后的密文
         */

        ByteBuffer bytebuffer = ByteBuffer.allocate( 4 + encryptedKey.length + 4 +encryptedData.length);
        bytebuffer.putInt(encryptedKey.length);
        bytebuffer.put(encryptedKey);
        bytebuffer.putInt(encryptedData.length);
        bytebuffer.put(encryptedData);

//        String base58encode = Base58.encode(bytebuffer.array());
//        System.out.println("base58 编码后的:   " + base58encode);

        // 进行 16 进制编码
        String hexencode = HexUtils.toHex(bytebuffer.array());

        System.out.println(" 将数字信封和密文组装后的报文 16进制格式：" + hexencode);

        System.out.println("发送方数据加密完成，可以将数据发送出去 ");

        /**
         *****************************************************  以下为接收方 代码  *************************************
         */

//        byte[] base58decode = Base58.decode(hexencode);
        byte[] hexdecode = HexUtils.toBytes(hexencode);
        ByteBuffer receiveBuffer = ByteBuffer.wrap(hexdecode);

        // 获取到对称秘钥长度
        int receivedEncryptedKeyLength = receiveBuffer.getInt();
        // 加密后的对称密钥key
        byte[] receivedEncryptKey = new byte[receivedEncryptedKeyLength];
        receiveBuffer.get(receivedEncryptKey,0,receivedEncryptedKeyLength);

        System.out.println(" 接收到的 加密后的对称密钥 ：" + HexUtils.toHex(receivedEncryptKey));
        // 获取到的 密文的长度
        int contextLength = receiveBuffer.getInt();
        // 密文
        byte[] receivedEncryptContext = new byte[contextLength];
        receiveBuffer.get(receivedEncryptContext,0,contextLength);

        System.out.println(" 接收到的 密文：" + HexUtils.toHex(receivedEncryptContext));


        // 使用接收方私钥，解密对称密钥
        byte[] receiveddecryptKey = null;
        try {
            receiveddecryptKey = ECCUtil.privateDecrypt(receivedEncryptKey,receiverECPrivateKey.getECPrivateKey());
        } catch (Exception e) {
            e.printStackTrace();
            System.out.println("  do something!!!!");
        }

        System.out.println(" 解密后的对称密钥 ：" + HexUtils.toHex(receiveddecryptKey));

        // 使用对称密钥，对密文进行解密

        try {
            byte[] plainText = CryptUtil.aesDecryptWithNOIV(receiveddecryptKey,receivedEncryptContext);
            // 解密后数据
            System.out.println("解密后数据 :  "+new String(plainText, "utf8"));
        } catch (InvalidCipherTextException e) {
            e.printStackTrace();
            System.out.println("  do something!!!!");
        }

```

## 根据 EOS 交易数据 恢复 公钥 示例

```java
        /**
         *  16 进制 交易数据
         */
        String hexData = "6a3f0e5f210a14b478050000000001603463937a4c8f440000b82a5d5a91d401809fd18c4dea8f4400000020476d964aa701809fd18c4dea8f440f495ab51e9cb44908424f53533130303008554d43503130303004554d43501f3533314249503242333235323032303034313531363238323231353336363908ab91465f12000006017560c95d040000007560c95d0400000000000000000000000100000000000000080006323130303130c818f4515f12000040035df1481b000006363938303237082d4359313647424e0200000000000000000000000000";
        /**
         *  eos 签名
         */
        String signStr = "SIG_K1_K1Pgvc9jXrCbPXx23zGugEfhxcGXoCSJjFEzFQj2HoFie18qnizWzQGssQezmS8PZ9fkKci3k8PGM2MQpJvuXtRZtd4oCD";
        EcSignature ecSignature = new EcSignature(signStr);
        Sha256 curData = getDigestForSignature(new TypeChainId("4a2fb7b7aacce5ea952dc96fbac6ed648efc08c1e1577882f3f33c82da248d64"),hexData);
        /**
         *  恢复公钥
         */
        EosPublicKey eosPublicKey =  EcDsa.recoverPubKey(curData.getBytes(), ecSignature);

        System.out.println("publicKey: " + eosPublicKey.toString());
        /**
         *  验证 恢复出来的公钥，与原公钥是否相同
         */
        Assert.assertEquals("EOS4wTJSTd29mZ4MNPZ2y4q2PtrimVpHLwHg8U4XMdzSBLa4BeYmN",eosPublicKey.toString());

```