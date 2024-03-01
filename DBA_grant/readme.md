- [密态权限——授权](#密态权限授权)
  - [创建密态表](#创建密态表)
  - [密态下用户授权](#密态下用户授权)
    - [实验步骤](#实验步骤)
    - [实验结论](#实验结论)
  - [注意](#注意)


# 密态权限——授权

## 创建密态表

- [openGauss 5.0.0全密态数据库应用小试-CSDN博客](https://blog.csdn.net/GaussDB/article/details/136184131)

1. 开启密态

```SQL
gsql -p 5432 -d test -U tom -r -C
```

2. 创建主密钥CMK，然后用它加密CEK，CEK是数据密钥，用于加密数据

   - 先创建CMK

   - ```sql
     CREATE CLIENT MASTER KEY ImgCMK WITH (KEY_STORE = localkms, KEY_PATH = "key_path_value", ALGORITHM = RSA_2048);
     ```

     创建成功会显示：CREATE CLIENT MASTER KEY

   - 再创建CEK

   - ```sql
     CREATE COLUMN ENCRYPTION KEY ImgCEK WITH VALUES (CLIENT_MASTER_KEY = ImgCMK, ALGORITHM = AEAD_AES_256_CBC_HMAC_SHA256);
     ```

     创建成功会显示：CREATE COLUMN ENCRYPTION KEY

3. 查看创建的密钥：

```sql
SELECT * FROM gs_client_global_keys;
```

<img src="密态权限.assets/image-20240301212835572.png" alt="image-20240301212835572" style="zoom:67%;" />

4. 创建加密表，不用连Database，可以直接创，\d可以直接查看

```sql
 CREATE TABLE creditcard_info (id_number int, name text encrypted with (column_encryption_key = ImgCEK, encryption_type = DETERMINISTIC),credit_card  varchar(19) encrypted with (column_encryption_key = ImgCEK, encryption_type = DETERMINISTIC));
```

<img src="密态权限.assets/image-20240301213001789.png" alt="image-20240301213001789" style="zoom:67%;" />

5. 插入数据即可，左为开启密态下，看到的是明文，退出密态模式查看，是密文形式

![image-20240301224638360](密态权限.assets/image-20240301224638360.png)

## 密态下用户授权

### 实验步骤

1. 登录lisa，在密态下，<u>看得到有这个表，但是禁止select查询</u>

```
gsql -p 5432 -d test -U lisa -r -C
```

<img src="密态权限.assets/image-20240301223852787.png" alt="image-20240301223852787" style="zoom:67%;" />

显示错误，权限禁止：

```
test=> select * from creditcard_info;
ERROR:  permission denied for relation creditcard_info
```

2. 启动密态，tom授权给lisa

```
GRANT SELECT ON creditcard_info TO lisa;
```

<img src="密态权限.assets/image-20240301214453131.png" alt="image-20240301214453131" style="zoom:67%;" />

3. 授权完毕后，另开一个终端，启动密态，查看lisa，<u>并进行select语句调用</u>

![image-20240301224142505](密态权限.assets/image-20240301224142505.png)

显示新的错误，无法进行解密：

```
test=> select * from creditcard_info;
ERROR(CLIENT): failed to decrypt column encryption key
```

4. 启动明文，登录lisa，并进行select语句调用

```
gsql -p 5432 -d test -U lisa -r
```

```
select * from creditcard_info ;
```

<img src="密态权限-grant.assets/image-20240301232850998.png" alt="image-20240301232850998" style="zoom:67%;" />

### 实验结论

- grant在密态下是成立的
- 被grant的用户：
  - 开启密态是看不到的，因为开启密态是要看明文，他没有相关密钥解密不了，所以报错
  - 开启明文是不会报错的，可以看到数据库内容的密文形式

## 注意

- 密钥存储在`/opt/og/openGauss-server/dest/etc/localkms/`下，四个为一组，可以删除

![image-20240301231403320](密态权限-grant.assets/image-20240301231403320.png)

CREATE CLIENT MASTER KEY ImgCMK WITH (KEY_STORE = localkms, KEY_PATH = "key_path_value", ALGORITHM = RSA_2048);——所以这个key_path_value是名字？
