# Alpha Saga 事务流转笔记

1. 正常的流转

![sequence-booking-normal](assets/sequence-booking-normal.png)

2. 服务异常

   HOTEL 服务内部异常，触发事务补偿

![sequence-booking-hotel-exception](assets/sequence-booking-hotel-exception.png)

3. 请求超时
   
   由于设置 request timeout 小于 Saga事务的 timeout 导致的请求超，步骤 7 晚于后台超时检查动作时
   
![sequence-booking-request-hotel-timeout](assets/sequence-booking-request-hotel-timeout.png)

4. 请求超时

   由于设置 request timeout 小于 Saga事务的 timeout 导致的请求超，步骤 6 早于后台超时检查动作时

![sequence-booking-request-hotel-timeout](assets/sequence-booking-request-hotel-timeout-2.png)

5. 请求超时

   由于设置 request timeout 小于 Saga事务的 timeout 导致的请求超，步骤 6 晚于步骤 5 发生时

![sequence-booking-request-hotel-timeout](assets/sequence-booking-request-hotel-timeout-3.png)

6. 事务超时

   Saga事务的 timeout 导致的超时

![sequence-request-hotel-timeout-after-txtimeout](assets/sequence-request-hotel-timeout-after-txtimeout.png)

7. 关于 EventScanner 粗略阅读了一遍，疑问暂时红色字体标出，后续再细读更新（以下描述如与上面的描述不同，则以上图为准）

![EventScanner_Reading](assets/EventScanner.png)