## 루틴 동작중 예기치 않게 멈추는 경ㅇ.

> 명확한 사유없이 루틴이 멈추는 경우, <br/>
> 일정시간마다 정지를 감지하여 재기동 <br/>
> (이지만, 근본적으로 멈추는 이유를 파악하는게 필요하다.) <br/>


```javascript
(function(){
    const schdObj = require('node-schedule');
    const orderObj = {

        // ...

        /**
        * 거래서버 정지시 조치 스케쥴러 시작
        */
        setRestartServerJob : function(){
            if(restartServerJob){
                logger.debug('restartServerJob is alreay running.');
                return;
            }

            // 스케쥴러전 마지막 거래루틴 시간 초기화
            orderObj.markLastOrderPrcsTime(0);

            // 스케쥴러 정의.
            restartServerJob = schdObj.scheduleJob('0 * * * * *',async ()=>{
                if(lastOrderPrcsTime < 1){
                    logger.debug('dont init orderServer.');
                    return;
                }

                // 바이낸스 서버 점검시간인 경우 거래 중지.
                if(orderObj.checkSystemMaintain()){
                    return ;
                }

                const curTimestamp = Date.now();
                const intervalTime = curTimestamp - lastOrderPrcsTime;
                const maxIntervalTime = 60 * 1000; // 최대 대기시간
                const waittingTime = (dealSet.intervalTime + 10);    // 강제종료후 재시작 대기시간(검사주기 + 10초)
                let waittingJob = null;

                // 60초이상 거래내역이 없는 경우, 서버정지로 간주.
                logger.debug('setRestartServerJob curInterval:'+intervalTime+'ms');
                if(intervalTime > maxIntervalTime){
                    logger.debug('setRestartServerJob orderServer is abnormal.[symbol:'+dealSet.symbol+', Af.Stop:'+intervalTime+'ms]');
                    logger.debug('and assumed that the server has stopped...so try restarting.');
                    
                    try{
                        errSvc.insertErrorCntn(jsonObj.getMsgJson('-1','[symbol:'+dealSet.symbol+'][stopInterval:'+intervalTime+']setRestartServerJob orderServer is abnormal. assumed that the server has stopped...so try restarting.'));
                        
                        // 1. 먼저 거래루틴 정지.
                        orderObj.stopOrdering();

                        // 2. (waittingTime)초후 거래루틴 재개.
                        waittingJob = schdObj.scheduleJob('0/'+waittingTime+' * * * * *', async ()=>{
                            try{
                                logger.debug(waittingTime+'sec has passed, prcessing restart retine...');
                                await orderObj.init();
                                orderObj.run();

                                let successMsg = 'setRestartServerJob orderServer restarting is success.';

                                sqlObj.insertSlackHistory(slackUtil.makeSlackMsgOfSingle(bConst.SLACK.TYPE.ERROR, slackUtil.setSlackTitle('Server restart'), successMsg));

                                logger.debug(successMsg);
                                errSvc.insertErrorCntn(jsonObj.getMsgJson('0',successMsg));

                            }catch(err){
                                logger.error('setRestartServerJob fail.');
                                errSvc.insertErrorCntn(jsonObj.getMsgJson('1',err));

                            }finally{
                                waittingJob.cancel();
                                waittingJob = null;
                            }
                        });

                    }catch(e){
                        errSvc.insertErrorCntn(jsonObj.getMsgJson('-1',objUtil.objView(e)));
                    }
                }else{
                    logger.debug('setRestartServerJob orderServer is normal.');
                }
            });
        }

        // ...

    }
})();


```