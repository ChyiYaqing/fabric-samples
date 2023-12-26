# Using EncCC

To test `EncCC` you need to first generate an AES 256 bit key as a base64
encoded string so that it can be passed as JSON to the peer chaincode
invoke's transient parameter.

Note: Before getting started you must use govendor to add external dependencies.  Please issue the following commands inside the "enccc_example" folder:
```
go mod vendor
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
```

* Install Hyperledger Fabric Samples

Navigate to the chaincode-docker-devmod directory of the fabric-samples clone:

```
$ cd chaincode-docker-devmode
```

* Terminal 1 - Start the network

```
$ docker-compose -f docker-compose-simple.yaml up
```

* Terminal 2 - Build & start the chaincode

```
$ docker exec -it chaincode bash
$ cd enccc_example
$ CORE_PEER_ADDRESS=peer:7052 CORE_CHAINCODE_ID_NAME=mycc:0 ./enccc_example
```

* Terminal 3 - Use the chaincode
Let's generate the encryption and decryption keys.  The example will simulate a shared key so the key is used for both encryption and decryption.
```
$ docker exec -it cli bash
# -n Name of the chaincode
# -v Version of the chaincode specified in install/instantiate/upgrade commands
$ peer chaincode install -p chaincodedev/chaincode/enccc_example -n mycc -v 0
$ peer chaincode instantiate -n mycc -v 0 -c '{"Args":[]}' -C myc

#ENCKEY=`openssl rand -base64 32` && DECKEY=$ENCKEY
ENCKEY=CSktUmyOfwIsn9aCeyZa7/9hLrM0gHeXKJdZ+Oo6C6c=
DECKEY=$ENCKEY

```

At this point, you can invoke the chaincode to encrypt key-value pairs as
follows:

Note: the following assumes the env is initialized and peer has joined channel "my-ch".
```
# -C -- channelID
peer chaincode invoke -n mycc -C myc -c '{"Args":["ENCRYPT","key1","value1"]}' --transient "{\"ENCKEY\":\"$ENCKEY\"}"
```

This call will encrypt using a random IV. This may be undesirable for
instance if the chaincode invocation needs to be endorsed by multiple
peers since it would cause the endorsement of conflicting read/write sets.
It is possible to encrypt deterministically by specifying the IV, as
follows: at first the IV must be created

```
IV=`openssl rand 16 -base64`
IV=Rrq9PTtTTQxoMVmnc32D8Q==
```

Then, the IV may be specified in the transient field

```
peer chaincode invoke -n mycc -C myc -c '{"Args":["ENCRYPT","key2","value2"]}' --transient "{\"ENCKEY\":\"$ENCKEY\",\"IV\":\"$IV\"}"
```

Two such invocations will produce equal KVS writes, which can be endorsed by multiple nodes.

The value can be retrieved back as follows

```
peer chaincode query -n mycc -C myc -c '{"Args":["DECRYPT","key1"]}' --transient "{\"DECKEY\":\"$DECKEY\"}"
```
```
peer chaincode query -n mycc -C myc -c '{"Args":["DECRYPT","key2"]}' --transient "{\"DECKEY\":\"$DECKEY\"}"
```
Note that in this case we use a chaincode query operation; while the use of the
transient(瞬态字段) field guarantees that the content will not be written to the ledger,
the chaincode decrypts the message and puts it in the proposal response. An
invocation would persist the result in the ledger for all channel readers to
see whereas a query can be discarded and so the result remains confidential.

To test signing and verifying, you also need to generate an ECDSA key for the appropriate
curve, as follows.

```
On Intel:
SIGKEY=`openssl ecparam -name prime256v1 -genkey | tail -n5 | base64 -w0` && VERKEY=$SIGKEY

On Mac:
SIGKEY=`openssl ecparam -name prime256v1 -genkey | tail -n5 | base64` && VERKEY=$SIGKEY
```

At this point, you can invoke the chaincode to sign and then encrypt key-value
pairs as follows

```
peer chaincode invoke -n enccc -C my-ch -c '{"Args":["ENCRYPTSIGN","key3","value3"]}' --transient "{\"ENCKEY\":\"$ENCKEY\",\"SIGKEY\":\"$SIGKEY\"}"
```

And similarly to retrieve them using a query

```
peer chaincode query -n enccc -C my-ch -c '{"Args":["DECRYPTVERIFY","key3"]}' --transient "{\"DECKEY\":\"$DECKEY\",\"VERKEY\":\"$VERKEY\"}"
```


### 扩展知识

* AES256 - 分组对称加密
```
AES256分组对称加密是指将铭文数据分解为多个16字节的明文块，利用密钥分别对每个明文块进行加密，得到相同个数的16字节蜜文块。

AES加密支持多种填充方式: NoPadding, PKCS5Padding, ZerosPadding, PKCS7Padding
    NoPadding: 表示不进行填充
    ZerosPadding: 表示用0进行填充缺少的位数
    PKCS5Padding,
```

* 跨平台编译
```
Macbook M2 架构 arm64, 为了编译出amd64位的执行文件

CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build
```
