# Mail Sending Schedule 信件寄送排程
### 1.需求  
由於BQool部分專案有些功能後續有綁定寄信通知需求,礙於功能需等待寄信流程結束才能繼續下一步.考量到執行等待時間因寄信而延長, 
加上架構上實踐單一職責原則盡量讓目的單一化.故特別把寄信功能移出各別專案作為獨立成定時排程.
### 2.服務端使用方式 
把所需寄信必備資訊(ex:mail_from,mail_to,Subject,context..etc) 寫入**Bqool_SetMain.Mail_Sending_Queue**.
再由寄信排程程式根據此**Mail_Sending_Queue**的**Send_Date**與**Status**來決定何時執行.
##### INSERT SQL語法 
```sql
INSERT INTO [dbo].[Mail_Sending_Queue]
([Account],[Category_ID],[Queue_Mode],[Mail_From],[Mail_To]
,[Mail_CC],[Mail_BCC],[Mail_Subject],[Mail_Context],[Is_Html]
,[Status],[Create_User] ,[Send_Date],[Create_Date],[Update_Date])
Values(
'Bqool_Account','PaymentConfirm',0,'do_not_reply@bqool.com','kondic@bqool.com'
,Null,Null,'Kondi mail test','<br>test context<br/>',1
,'I','kondi_user',GETUTCDATE(),GETUTCDATE(),GETUTCDATE()
)
```
Mail_Sending_Queue欄位要填什麼資料,請參考 [4.資料表欄位敘述](#4.資料表欄位敘述)
### 3.流程圖
```flow
st=>start: 服務端寫入信件資訊到DB.Queue
opGet=>operation: 取狀態為I且寄信時間已符合的DB.Queue,並更新其狀態為P
opMail=>operation: 寄信(SMTP or MailGun)
opSaveSys=>operation: 寫入SaveSystemMail
opSetStatus=>operation: 更新狀態 失敗:F 成功:D
opUpdateHis=>operation: 若Queues都跑完,整理失敗的queue並寄警告信,最後寫入DB.Queue_History,
condSaveSys=>condition: 寫入SaveSystemMail?
condAny=>condition: 有未寄信的Queue?

e=>end: 結束

st->opGet->opMail->condSaveSys
condSaveSys(yes)->opSaveSys->opSetStatus
condSaveSys(no)->opSetStatus->opUpdateHis->e
```
### 4.資料表欄位敘述
#### Bqool_SetMain.Mail_Sending_Queue
寄信排程主要取queue的table, 在排程完成寫入Queue_history後便刪除此table已執行的queue(status='p')

欄位名稱  | 資料型態 |NOT NULL|敘述
------------- | -------------||
Queue_ID 🔑|int|NOTNULL|此table的主鍵,用於區別重複資料與紀錄
Account|varchar(15)|NOTNULL|Bqool客戶帳號
Category_ID|varchar(50)|NOTNULL|此信件的用途分類 ex: MailConfm,PaymentConfirm
Queue_Mode|tinyint|NOTNULL|目前排程模式 0:寄信與寫入SaveSystemMail, 1:僅寄信
Mail_From|nvarchar(50)|NULL|寄信來源
Mail_To|nvarchar(200)|NOTNULL|收信來源
Mail_CC|nvarchar(200)|NULL|副本發送來源
Mail_BCC|nvarchar(200)|NULL|密件副本發送來源
Mail_Subject|nvarchar(500)|NOTNULL|信件標題
Mail_Context|nvarchar(max)|NOTNULL|信件內容
Is_Html|bit|NOTNULL|內容是否為html
Status|varchar(1)|NOTNULL|Queue狀態 I為初始狀態,P為正在執行
Create_User|varchar(50)|NOTNULL|客戶端使用者
Send_Date|datetime|NOTNULL|執行寄信時間
Create_Date|datetime|NOTNULL|產生時間
Update_Date|datetime|NOTNULL|更新時間

#### Bqool_SetMain.Mail_Sending_Queue_History
作為已執行queue後的寄信資訊紀錄用,欄位完全相同.但要注意此table沒有主鍵,且status也僅記錄執行後失敗的'F'或是成功的'D'

欄位名稱  | 資料型態 |NOT NULL|敘述
------------- | -------------||
Queue_ID |int|NOTNULL|用於區別重複資料與紀錄,非此table主鍵
Status|varchar(1)|NOTNULL|Queue狀態 F為失敗,D為成功
*僅列敘述不同的欄位,其他欄位參考Mail_Sending_Queue

### 5.排程組態描述
####appSettings
KEY  | 敘述
------------- | -------------
IsMailGun  | 是否使用MailGun ,預設為true
MailGunDomain  |Bqool註冊MailGun的Domain
MailGunKey|Bqool在MailGun公鑰
AlterMailto|警告信收信對象
AlertMailFrom|警告信來源對象
ReSendTimes|寄信失敗重複寄信最大次數 ,預設為三次
ReSendSleepSeconds|重複寄信中間執行序等待時間(秒) ,預設一秒

### 6.其他補充

開發完成日期:10/21/2021 by Kondi C
