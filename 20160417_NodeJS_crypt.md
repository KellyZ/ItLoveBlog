## web系列005：NodeJS crypt模块
NodeJS原生Crypto模块提供基本的加解密功能。

## 接口
1. Class: Certificate
2. Class: Cipher
3. Class: Decipher
4. Class: DiffieHellman
5. Class: ECDH
6. Class: Hash
7. Class: Hmac
8. Class: Sign
9. Class: Verify
10. crypto.DEFAULT_ENCODING
11. crypto.createCipher(algorithm, password)
12. crypto.createCipheriv(algorithm, key, iv)
13. crypto.createCredentials(details)
14. crypto.createDecipher(algorithm, password)
15. crypto.createDecipheriv(algorithm, key, iv)
16. crypto.createDiffieHellman(prime[, prime_encoding](#)[,generator](#)[, generator_encoding](#))
17. crypto.createDiffieHellman(prime_length[, generator](#))
18. crypto.createECDH(curve_name)
19. crypto.createHash(algorithm)
20. crypto.createHmac(algorithm, key)
21. crypto.createSign(algorithm)
22. crypto.createVerify(algorithm)
23. crypto.getCiphers()
24. crypto.getCurves()
25. crypto.getDiffieHellman(group_name)
26. crypto.getHashes()
27. crypto.pbkdf2(password, salt, iterations, keylen[, digest](#), callback)
28. crypto.pbkdf2Sync(password, salt, iterations, keylen[, digest](#))
29. crypto.privateDecrypt(private_key, buffer)
30. crypto.privateEncrypt(private_key, buffer)
31. crypto.publicDecrypt(public_key, buffer)
32. crypto.publicEncrypt(public_key, buffer)
33. crypto.randomBytes(size[, callback](#))
34. crypto.setEngine(engine[, flags](#))

## 网络安全模型
在网络上通信通常会有两类安全问题：
1. 身份认证
2. 数据安全：保密性、完整性

### 身份认证
常用网络情况：
1. 服务器提供公开服务，需要证明服务器的身份，如支付宝、银行等；
2. 客户端API接口通信服务端。

身份认证依赖密码学和共享秘密：如果A、B通信双方共享一个秘密，即它们两个知道秘密但分布式系统中的其他通信端不知道这个秘密。如果由一对进程交换的消息包括**证明**发送方共享秘密的信息，那么接收方就能确认发送方。当然，必须小心以确保共享的秘密不泄露给敌人。

那共享秘密如何被认证呢？基本的认证技术是在消息中包含加密部分，该部分中包含足够的消息内容以保证它的真实性，其认证部分使用**共享的密钥**加密。密码学中使用**数字签名**。

#### 数字签名
**数字签名**，就是只有信息的发送者才能产生的别人无法伪造的一段数字串，这段数字串同时也是对信息的发送者发送信息真实性的一个有效证明。

数字签名是**非对称密钥加密技术**与**数字摘要技术**的应用，一套数字签名通常定义两种互补的运算，一个用于签名，另一个用于验证。

**签名**：发送报文时，发送方用一个哈希函数从报文文本中生成报文摘要,然后用自己的私钥对这个摘要进行加密，这个加密后的摘要将作为报文的数字签名和报文一起发送给接收方

**验证**：接收方首先用与发送方一样的哈希函数从接收到的原始报文中计算出报文摘要，接着再用发送方的公用密钥来对报文附加的数字签名进行解密，如果这两个摘要相同、那么接收方就能确认该数字签名是发送方的

签名用私钥，验证用公钥。

数字签名有两种功效：一是能确定消息确实是由发送方签名并发出来的，因为别人假冒不了发送方的签名。二是数字签名能确定消息的完整性。

普通数字签名算法有RSA、ElGamal、Fiat-Shamir、Guillou- Quisquarter、Schnorr、Ong-Schnorr-Shamir数字签名算法、Des/DSA,椭圆曲线数字签名算法
**NodeJS实现签名**
	var crypto = require('crypto');
	var fs = require('fs');
	
	var sign = crypto.createSign('RSA-SHA256');
	var verify = crypto.createVerify('RSA-SHA256');
	
	var privateKey = fs.readFileSync('./rsa/ca.key').toString(); 		//rsa私钥
	var publicKey = fs.readFileSync('./rsa/ca.crt').toString();		//rsa公钥
	
	var s = fs.ReadStream('./file1');
	s.on('data', function(d) {
	  sign.update(d);
	  verify.update(d);
	});
	
	s.on('end', function() {
	var signture = sign.sign(privateKey);//生成签名
	var result = verify.verify(publicKey, signture);//验证签名
	console.log(result);	//true
	});

**NodeJS实现RSA**
NodeJS crypt模块没有支持RSA，需要第三方ursa库实现，ursa底层使用openssl。
	var ursa = require("ursa");
	
	var clearText='使用RSA加密解密字符串';
	
	var keySizeBits = 1024;
	var keyPair = ursa.generatePrivateKey(keySizeBits, 65537);
	
	var encrypted = encrypt(clearText, keySizeBits/8);
	console.log('加密后的数据：'+encrypted);
	
	var decrypted = decrypt(encrypted, keySizeBits/8);
	console.log('加密数据解密：'+decrypted);
	
	function encrypt(clearText, keySizeBytes){
	var buffer = new Buffer(clearText);
	var maxBufferSize = keySizeBytes - 42; //according to ursa documentation
	var bytesDecrypted = 0;
	var encryptedBuffersList = [];
	
	//loops through all data buffer encrypting piece by piece
	while(bytesDecrypted < buffer.length){
	//calculates next maximun length for temporary buffer and creates it
	var amountToCopy = Math.min(maxBufferSize, buffer.length - bytesDecrypted);
	var tempBuffer = new Buffer(amountToCopy);
	
	//copies next chunk of data to the temporary buffer
	buffer.copy(tempBuffer, 0, bytesDecrypted, bytesDecrypted + amountToCopy);
	
	//encrypts and stores current chunk
	var encryptedBuffer = keyPair.encrypt(tempBuffer);
	encryptedBuffersList.push(encryptedBuffer);
	
	bytesDecrypted += amountToCopy;
	}
	
	//concatenates all encrypted buffers and returns the corresponding String
	return Buffer.concat(encryptedBuffersList).toString('base64');
	}
	
	function decrypt(encryptedString, keySizeBytes){
	
	var encryptedBuffer = new Buffer(encryptedString, 'base64');
	var decryptedBuffers = [];
	
	//if the clear text was encrypted with a key of size N, the encrypted 
	//result is a string formed by the concatenation of strings of N bytes long, 
	//so we can find out how many substrings there are by diving the final result
	//size per N
	var totalBuffers = encryptedBuffer.length / keySizeBytes;
	
	//decrypts each buffer and stores result buffer in an array
	for(var i = 0 ; i < totalBuffers; i++){
	//copies next buffer chunk to be decrypted in a temp buffer
	var tempBuffer = new Buffer(keySizeBytes);
	encryptedBuffer.copy(tempBuffer, 0, i*keySizeBytes, (i+1)*keySizeBytes);
	//decrypts and stores current chunk
	var decryptedBuffer = keyPair.decrypt(tempBuffer);
	decryptedBuffers.push(decryptedBuffer);
	}
	
	//concatenates all decrypted buffers and returns the corresponding String
	return Buffer.concat(decryptedBuffers).toString();
	}

#### 数字摘要
所谓的消息摘要就是通过一种单向算法计算出来的唯一对应一个文件或数据的固定长度的值，也被称作数字摘要。根据不同的算法，消息摘要的长度一般为128位或160位。常用的消息摘要的算法有MD5和SHA1。一个文件的消息摘要就同一个人的指纹一样，它可以唯一代表这个文件，如果这个文件被修改了，那么它的消息摘要也一定会发生变化，所以，我们可以通过对一个文件的**消息摘要进行签名**来代替对它本身进行签名。
**MD5实现**
	var crypto = require('crypto');
	var content = 'password’；
	var md5 = crypto.createHash('md5');
	md5.update(content);
	var d = md5.digest('hex'); 
**SHA实现**
	var crypto = require('crypto');
	var content = 'password’；
	var shasum = crypto.createHash('sha1');
	shasum.update(content);
	var d = shasum.digest('hex');
随着互联网的发展，MD5已经变得越来越不安全了，黑客可以通过彩虹表，查出MD5值所对应的密码，为了解决这个问题，很多网站都开始采用需要密钥加密的Hmac算法。
**Hmac实现**
	const crypto = require('crypto');
	const hmac = crypto.createHmac('sha256', 'a secret');
	
	hmac.update('some data to hash');
	console.log(hmac.digest('hex'));

hmac主要应用在身份验证中，它的使用方法是这样的：
(1) 客户端发出登录请求（假设是浏览器的GET请求）
(2) 服务器返回一个随机值，并在会话中记录这个随机值
(3) 客户端将该随机值作为密钥，**用户密码**进行hmac运算，然后提交给服务器
(4) 服务器读取用户数据库中的**用户密码**和步骤2中发送的随机值做与客户端一样的hmac运算，然后与用户发送的结果比较，如果结果一致则验证用户合法

在这个过程中，可能遭到安全攻击的是服务器发送的随机值和用户发送的hmac结果，而对于截获了这两个值的黑客而言这两个值是没有意义的，绝无获取用户密码的可能性，随机值的引入使hmac只在当前会话中有效，大大增强了安全性和实用性。大多数的语言都实现了hmac算法，比如php的mhash、python的hmac.py、java的MessageDigest类，在web验证中使用hmac也是可行的，用js进行md5运算的速度也是比较快的。

由上面的介绍，我们可以看出，HMAC算法更象是一种加密算法，它引入了密钥，其安全性已经不完全依赖于所使用的HASH算法

### 数据安全
数据加密使用对称加密，对称加密挑战问题是密钥的安全，解决密钥安全通过密钥交换实时生成。
**Diffie-Hellman实现**
	var crypto = require('crypto');
	
	var alice = crypto.getDiffieHellman('modp5');
	var bob = crypto.getDiffieHellman('modp5');
	
	//也可以使用如下方式生成创建对象
	// var alice = crypto.createDiffieHellman(7);
	// var bob = crypto.createDiffieHellman(7);
	
	//生成公钥和私钥
	alice.generateKeys();
	bob.generateKeys();
	
	//计算共享密钥相同
	var alice_secret = alice.computeSecret(bob.getPublicKey(), null, 'hex');
	var bob_secret = bob.computeSecret(alice.getPublicKey(), null, 'hex');
	
	//两个共享密钥相同
	console.log(alice_secret == bob_secret);	//true
**EC Diffie-Hellman**
	var crypto = require('crypto');
	
	var alice = crypto.createECDH('secp256k1');
	var bob = crypto.createECDH('secp256k1');
	
	//生成公钥和私钥
	alice.generateKeys();
	bob.generateKeys();
	
	//计算共享密钥相同
	var alice_secret = alice.computeSecret(bob.getPublicKey(), null, 'hex');
	var bob_secret = bob.computeSecret(alice.getPublicKey(), null, 'hex');
	
	//两个共享密钥应该相同
	console.log(alice_secret == bob_secret);    //tre
**对称加密**
	var crypto = require('crypto');
	var fs = require('fs');
	
	var cipher = crypto.createCipher('aes192', new Buffer('my password'));
	var decipher = crypto.createDecipher('aes192', new Buffer('my password'));
	
	var s = fs.ReadStream('./file1');
	s.on('data', function(d) {
	  cipher.update(d);
	});
	
	s.on('end', function() {
	  var d = cipher.final();
	  console.log('加密后的数据是：%s', d.toString('hex'));
	  decipher.update(d);
	  console.log('解密后的数据是：%s', decipher.final().toString());
	});

## 参考
1. [http://baike.baidu.com/view/7626.htm](http://baike.baidu.com/view/7626.htm)
2. [http://blog.shiqichan.com/encrypt-and-decrypt-string-with-rsa/](http://blog.shiqichan.com/encrypt-and-decrypt-string-with-rsa/)
3. [http://me2xp.blog.51cto.com/6716920/1535027](http://me2xp.blog.51cto.com/6716920/1535027)
4. [http://www.aireadfun.com/blog/2014/01/01/application-scenarios-of-rsa-and-nodejs-ursa-toturial/](http://www.aireadfun.com/blog/2014/01/01/application-scenarios-of-rsa-and-nodejs-ursa-toturial/)
5. [http://itbilu.com/nodejs/core/NkQOETLK.html](http://itbilu.com/nodejs/core/NkQOETLK.html)
6. [http://itbilu.com/nodejs/core/EJOj6hBY.html](http://itbilu.com/nodejs/core/EJOj6hBY.html)
7. [https://nodejs.org/api/crypto.html](https://nodejs.org/api/crypto.html)