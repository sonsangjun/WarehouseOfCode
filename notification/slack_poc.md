## Node.js기반 적용할 Slack

> DB에 이력을 쌓기전에 오직 Slack을 위한 서비스 코드 <br/>

```javascript

const dealSetSvc = require('./dealSetService');
const jsonUtil = require('../util/jsonUtil');
const logger = require('../conf/winston');
const objUtil = require('../util/objectUtil');

const { WebClient } = require('@slack/web-api');

const dotenv = require('dotenv');

dotenv.config();

module.exports = (function(){
    const jsonObj = jsonUtil.getJsonObj('staticsService');
    const isDev = objUtil.checkDevMode();
    const symbol = process.env.targetSymbol;
    
    let slackObj = null;
    let svcObj = {};
    let isInit = false;
    let dealSet = dealSetSvc.getDefaultSet();

    /**
     * Slack초기화
     */
    svcObj.initSlack = function(){
        return (new Promise(async (resolve, reject)=>{
            try{
                const result = await dealSetSvc.selectDealSetFromDBnEnv();
                isInit = true;
                dealSet = result;
                
                slackObj = new WebClient(dealSet.slack.token);

                resolve(jsonObj.getMsgJson('0','initDealSet success.'));

            }catch(err){
                logger.error('initSlack fail. '+objUtil.objView(err));  
                reject(jsonObj.getMsgJson('-1', err));
            }
        }));
    };

    /**
     * 초기화 여부 체크
     */
    svcObj.checkInit = function(){
        return isInit;
    };

    svcObj.sendMsgInSlackWithEvent = function(title, message){
        const cnvttitle = svcObj.viewTitleOfDev() + '['+symbol+':EVENT]\n# CurDate: '+svcObj.getFullTimeWithDelimter(Date.now())+'\n# '+title+'\n\n';
        return svcObj.sendMsgInSlack(cnvttitle + message);    
    },

    svcObj.sendMsgInSlackWithError = function(title, message){
        const cnvttitle = svcObj.viewTitleOfDev() + '['+symbol+':ERROR]\n# CurDate: '+svcObj.getFullTimeWithDelimter(Date.now())+'\n# '+title+'\n\n';
        return svcObj.sendMsgInSlack(cnvttitle + message);    
    },

    svcObj.getFullTimeWithDelimter = function(curTime){
        return ''.concat(objUtil.getYYYYMMDD(curTime),'.',objUtil.getHHMMSS(curTime));
    }

    /**
     * 타이틀 앞에 붙는 DEV결정
     */
    svcObj.viewTitleOfDev = function(){
        return (isDev?'[DEV]':'');
    },

    /**
     * 슬랙 메시지 전송
     */
    svcObj.sendMsgInSlack = function(message){
        return (new Promise(async (resolve, reject)=>{
            try{
                if(!svcObj.checkInit()){
                    logger.error('isInit is fail. start init.');  
                    await svcObj.initSlack();
                }

                if(!(dealSet.slack && dealSet.slack.token)){
                    return reject(jsonObj.getMsgJson('-1','slackService token is empty.'));    
                }

                const result = await slackObj.chat.postMessage({
                    text: message,
                    channel: dealSet.slack.channel,
                });
            }catch(err){
                logger.error('initSlack fail. '+objUtil.objView(err));  
                reject(jsonObj.getMsgJson('-1', e));
            }
        }));
    };
    
    return svcObj;
})();

```