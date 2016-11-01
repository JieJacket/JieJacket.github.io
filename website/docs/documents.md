
### 快速体验

### 快速集成

* 导入 SDK 到你的项目中

* 导入资源文件

* 修改 build.gradle 配置文件


### 连接POS机

使用 N900 智能 POS 机器, 我们的应用首先需要连接 POS 机硬件, 使用 SDK 提供的 `connect` 方法连接。
建议在 `Application` 的 `onCreate` 方法中进行连接。

```java
    @Override
    public void onCreate() {
        super.onCreate();
        
        ...
        CILSDK.setDebug(true);//发版时请改为 false
        CILSDK.connect(this);
        
        ...
    }
```

### 激活POS机

第一次使用智能 POS 终端的时候,需要使用讯联下发的商户号 `merCode` 和终端号 `termCode` 激活,
激活成功之后才能正常使用后面的交易流程。

> 注意:一台终端能且仅能成功激活一次,无法重复进行激活操作。若有多次激活的需求(如debug阶段),请联系讯联客服。

```java
    //merCode  讯联下发的商户号
    //termCode 讯联下发的终端号
    CILSDK.active(merCode, termCode, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            
            if (cilResponse.getStatus() == 0) {
                //激活成功
            } else {
                //激活失败
            }
        }

        @Override
        public void onError(Parcelable p, Exception e) {
            //激活出错
        }
    });
```


### 终端参数下载

激活成功之后,你的应用还需要下载一些交易时使用的参数,比如`交易地址和端口`、`交易超时时间`、`终端支持的功能`、`TPDU` 等,
在你拿到 POS 终端之前,这些参数都会在讯联后台已经配置好,全部参数见 `CILResponse.Info` 返回值。下载成功之后 SKD 会
以json字符串的形式保存这些参数到 `SharedPreference` (请不要擅自改动这些参数值,以免导致交易失败) 。当然,在 `onResult`
中也会返回,你也可以自己选择性保存一些参数。另外, SDK 保存在 SharedPreference 里的值会提供接口 `CILSDK.getSystemParams()` 获取。

```java
    //merCode  讯联下发的商户号 激活成功之后可以确定商户号
    //termCode 讯联下发的终端号 激活成功之后可以确定终端号
    CILSDK.downloadParams(merCode, termCode, new com.cardinfolink.pos.listener.Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
            if (0 == response.getStatus()) {
                //参数下载成功,具体返回的参数见 response.Info
            } else {
                //参数下载错误
            }
        }

        @Override
        public void onError(Parcelable p, Exception e) {
            //下载出错
        }
    });
```

### 终端密钥下载

终端密钥下载只需要成功执行一次就可以,成功下载的密钥会被转载到POS硬件模块里面,后面就不需要再次调用了,建议你的应用可以在成功下载密钥之后持久化一个标志位,
下次进入应用就不再去下载密钥了。整个过程可能会需要1~2分钟左右(依赖当前的网络状况),会经历以下步骤:

> 请求讯联网关 RSA -> 装载 RSA -> 请求主密钥 -> 装载主密钥 -> 启用主密钥 -> 请求工作密钥(签到) -> 装载工作密钥 -> 下载 AID -> 装载 AID -> 下载 IC 公钥 -> 装载 IC 公钥

```java
    CILSDK.downloadParamsWithProgress(new ProgressCallback<CILResponse>() {
        @Override
        public void onProgressUpdate(int progress) {
            //progress下载密钥的进度
        }

        @Override
        public void onResult(CILResponse response) {
            
            if (0 == response.getStatus())
                //密钥下载成功。在这里可以持久化一个标志位
        }

        @Override
        public void onError(Parcelable p, Exception e) {
            //下载密钥出错
        }
    });
```


### 签到

签到其实也就是更新工作密钥的一个过程 (`下载工作密钥+装载工作密钥` ),讯联网关平台要求应用需要每天签到一次。

```java
    CILSDK.signIn(new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse cilResponse) {
            //签到成功
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            //签到出错
        }
    });
```


### 发起交易

SDK 将交易分为`银行卡交易`(刷卡、插卡和挥卡)和`扫码交易`(支付宝和微信)两种。银行卡相关交易需要POS机硬件模块读卡器,扫码相关交易则不需要。

#### a、银行卡交易

由于银行卡交易逻辑有点复杂,讯联提供了一个 `BaseCardActivity` 基础类,你只需要继承这个类便可以做银行卡类的交易了。具体使用方法可以见 demo 
里的 `CommonCardHandlerActivity` 类。下面给个大概说明:

```java
    public class CommonCardHandlerActivity extends BaseCardActivity {
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            
            ...
            
        }
        
        /**
         * 必须传入金额
         *
         * @return
         */
        public abstract String getAmount() {
            //在这里传入金额
        }
    
        /**
         * 读卡的结果
         *
         * @param isSuccess 是否成功
         * @param cardType  卡片种类
         * @param cardInfo  读取卡片信息
         */
        public abstract void cardReaderHandler(boolean isSuccess, @CardType.Type int cardType, CardInfo cardInfo){
            //在这里面发送银行卡相关的交易(如消费、消费撤销、退货、预授权、预授权撤销、预授权完成、预授权完成撤销、余额查询)
            //CILSDK.consume(request, cardType, new Callback<CILResponse>() //消费
        }
   
        /**
         * 显示读卡时的缓冲页面
         */
        public abstract void waitLoadingShow(){
        
        }
    
        /**
         * 取消读卡时的缓冲页面
         */
        public abstract void waitLoadingDismiss(){
        
        }
    
        /**
         * 读卡失败
         */
        public abstract void cardHandlerError(Exception e){
        
        }
            
    }
```

#### b、扫码交易

扫码相关的交易则是不依赖 POS 机器的读卡模块的,但是你需要将`微信`或者`支付宝`的二维码读出来传给扫码消费接口,扫码可以使用第三方库,如 `zxing`。

```java
    /**
     * 扫码消费
     */
    ...
    request.setAmount(amount);
    request.setScanCodeId(result);//二维码code
    
    CILSDK.consumeQr(request, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
            if (response.getTrans() != null && TextUtils.equals(response.getTrans().getRespCode(), "00")) {
                //扫码消费成功
            } else {
                //扫码消费失败
            }
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            //扫码消费出错
        }
    });
    
    /**
     * 扫码撤销
     */
    CILSDK.revokeConsumeQr(request, new Callback<CILResponse>()
    
    /**
     * 扫码退货
     */
    CILSDK.returnConsumeQr(request, new Callback<CILResponse>()
```

### 账单查询

智能 POS SDK 分别提供了近7天的`账单列表查询`和`账单统计接口`接口,接口会根据 type 值确定返回`银行卡账单(0)`或`扫码账单(1)`。

```java
    /**
     * 获取账单列表 异步
     *
     * @param page 从0开始
     * @param size 每页返回的条数
     * @param type 账单类型
     * <ul>
     *      <li>TransConstants.CARD_BILL:银行卡账单</li>
     *      <li>TransConstants.QR_BILL:扫码账单</li>
     * </ul>
     *
     */
    CILSDK.getBillsAsync(page, size,@BillType int txnType, new Callback<CILResponse>() {
        @Override
        public void onResult(final CILResponse response) {
            if (null != response && 0 == response.getStatus()) {
                //账单获取成功
                Trans[] trans = response.getTxn(); //账单数据,字段详情见 Trans
            }
        }

        @Override
        public void onError(Parcelable p, Exception ex) {
            //账单获取出错
        }
    });
    
    /**
     * 获取账单统计信息 异步
     * @param type 账单类型
     * <ul>
     *      <li>TransConstants.ALL_BILL:所有账单</li>
     *      <li>TransConstants.CARD_BILL:银行卡账单</li>
     *      <li>TransConstants.QR_BILL:扫码账单</li>
     * </ul>
     *
     */
    CILSDK.getBillStatAsync(@BillType int txnType, new Callback<CILResponse>() {
        @Override
        public void onResult(CILResponse response) {
            if (null != response && 0 == response.getStatus()) {
                //账单获取成功
            }

        }

        @Override
        public void onError(Parcelable p, Exception ex) {
            //账单获取出错
        }
    });
    
```

> SDK 的网络部分使用的是第三方库 `okhttp`,以上账单接口分别还提供了相对应的同步接口 `getBills` 和 `getBillStat` 。
对于异步接口来说,都会返回一个 `Call` 对象,你可以在应用出错的时候调用 `call.cancel()` 取消这次请求,以免造成内存泄露。

### 结算

结算需求主要用于每日交易结束时或收银员交接班时,对某段时间内的账款核对。商户每日交易结束后,收银员需要统计并核对所有的交易,核对交易统计准确后结算，打印出结算单。
结算会涉及到一个概念--`批次号`,我们在前面的交易都会传入一个批次号给 request ,调用结算之后,后续的交易需要将这个批次号加1,因为此批次已经打包结算掉了。

```java
    //batchNum 批次号
    CILSDK.transSettleAsync(batchNum, new Callback<CILResponse>() {
        @Override
        public void onResult(final CILResponse response) {
            //打印结算单
        }

        @Override
        public void onError(Parcelable cilRequest, Exception e) {
            //结算出错
        }
    });
```

### 打印
本模块可用于根据交易信息打印所需的消费票据。接口不仅提供了一套固定格式的小票样式，而且还可以根据需要自定义打印样式。
主要功能包括小票打印、二维码打印、条形码打印以及图片打印。

* 打印银行卡类交易、扫码类交易小票
```java
    /** trans 交易信息
     *  lineBreak 小票结尾需要走纸换行的行数
     *  formatTransCode @FormatTransCode String类型，小票的交易类型
     *  kind @ReceiptSubtitle int类型，小票的子标题，判断是商户联或者是客户联
     *  isForeignTrans 是否是外卡类交易
     */
    CILSDK.printKindsReceipts(trans,lineBreak,formatTransCode, kind, isForeignTrans,  new Callback<PrinterResult>(){
         @Override
         public void onResult(PrinterResult response) {
             if (null ！= printerResult && !"打印成功".equals(printerResult.toString())) {
                 //打印成功
             }
         }
           
          @Override
          public void onError(Parcelable cilRequest, Exception e) {
                 //打印失败
          }
    }); 
```

* 打印结算小票

```java
    /**
     * 打印结算小票
     *
     * transSettles    结算信息List
     * transDatetime   结算时间
     * lineBreak       打印结尾换行数
     * formatTransCode 结算类型；TransConstants.TRANS_SETTLE_DETAILS：结算详情小票；TransConstants.TRANS_SETTLE_TOTAL：结算统计小票
     * callback 回调
     */
     CILSDK.printSettleReceipts(transSettles, transDatetime,batchNum, lineBreak, formatTransCode, new Callback<PrinterResult>() {
          @Override
          public void onResult(PrinterResult result) {
              if (null != result && !"打印成功".equals(result.toString())){
                                           
               }
          }    
          @Override
          public void onError(Parcelable cilRequest, Exception e) {
                                       
          }
     });

```

* 自定义打印

```java
    /**
     * 根据二进制数据打印(根据打印规范用户自定义打印小票样式)
     *
     * buffer    打印内容
     * lineBreak 换行数
     * callback  回调
     */
   CILSDK.printBufferReceipt(buffer, lineBreak,new Callback<PrinterResult>() {
        @Override
        public void onResult(PrinterResult result) {
            if (null != result && !"打印成功".equals(result.toString())){
                                                                                                                                         
             }
        }    
        @Override
        public void onError(Parcelable cilRequest, Exception e) {
                                                                                                                                     
        }
    });

```

* 打印二维码

```java
    /**
     * 打印二维码
     *
     * qrCode   二维码内容
     * position 打印位置  0:左对齐；1居中；2：右对齐
     * width    二维码宽度
     * callback 回调
     *
     */
    CILSDK.printQRCode(qrCode,position,width,lineBreak,new Callback<PrinterResult>() {
         @Override
         public void onResult(PrinterResult result) {
             if (null != result && !"打印成功".equals(result.toString())){
                                                                                                  
             }
         }    
         @Override
         public void onError(Parcelable cilRequest, Exception e) {
                                                                                              
         }
    });
```

* 打印条形码

```java
    /**
     * 打印条形码
     *
     * barCode   条形码数字
     * position  条形码位置 0:左对齐；1居中；2：右对齐
     * lineBreak 换行数
     * callback  回调
     */
    CILSDK.printBarCode(String barCode, int position, int lineBreak, new Callback<PrinterResult>() {
         @Override
         public void onResult(PrinterResult result) {
             if (null != result && !"打印成功".equals(result.toString())){
                                                                                                                                                                                                                        
              }
         }    
         @Override
         public void onError(Parcelable cilRequest, Exception e) {
                                                                                                                                                                                                                    
         }
    });
```

* 打印图片

```java
    /**
     * 打印图片
     *
     * bitmap    图片Bitmap
     * lineBreak 换行数
     * offset    偏移量
     * callback  回调
     *
     */
    CILSDK.printImage(bitmap, lineBreak, offset, new Callback<PrinterResult>() {
         @Override
         public void onResult(PrinterResult result) {
             if (null != result && !"打印成功".equals(result.toString())){
                                                                                                                                                   
             }
         }    
         @Override
         public void onError(Parcelable cilRequest, Exception e) {
                                                                                                                                               
         }
    });
```


### 其他设置

考虑到使用 SDK 的时候可能还会有其他需求,比如`获取 POS 机的 SN 号`、`设置密钥索引`等,在这里,我们也提供了一部分接口。

* 获取 SDK 版本

```java
    //版本名
    String versionName = CILSDK.VERSION_NAME;
    //版本号
    int versionCode = CILSDK.VERSION_CODE;
```

* 获取 SN 号

```java
    //SN号
    String snCode = CILSDK.getDeviceSN();
```

* 设置流水号

```java
    //serialNum范围：1~999999
    boolean isSuccess = CILSDK.setSerialNum(int serialNum);
```

* 获取流水号

```java
    //序列号
    int serialNum = CILSDK.getSerialNum();
```

* 设置批次号

```java
    //batchNum范围：1~999999
    boolean isSuccess = CILSDK.setBatchNum(int batchNum);
```

* 获取批次号

```java
    //批次号
    int batchNum = CILSDK.getBatchNum();
```

* 设置密钥索引


### 许可证

Copyright (c) 2016 cardinfolink.com



