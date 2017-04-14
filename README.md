# Clearwater-SIPp-stress-test
Stress test with Clearwater project.


##STEP1:

環境架設:

先依照Clearwater Manual Install 架設一台Clearwater的元件請參閱[clearwater Manual Install Guide](http://clearwater.readthedocs.io/en/stable/Manual_Install.html )  
>在設定local_config和shared_config時 所有要設定的IP都打自己host IP  
>Hostname要打你準備進行壓力測試的clearwater的realm

將其中Install Node-Specific Software階段的指令改為
    #sudo apt-get install clearwater-sip-stress    
clearwater-management不用裝  
然後你就安裝好SIPp了  
>此時/usr/share/clearwater/sip-stress/會有2個檔案: sip-stress.xml & users.csv.1     

>這裡可以打開users.csv.1來看 所有帳號的密碼都是7kkzTyGW  
>所以等一下在homestead建帳號的時候 密碼都要改成這個密碼  
>並且要注意裡面的realm是否跟存在homestead的realm一樣  

##STEP2:

此步驟要開始在clearwater的HSS(homestead)存壓力測試的user account  

*以下步驟皆在homestead host 內完成* 

1.於user目錄執行此指令建立測試帳號  

./etc/clearwater/config; for DN in {2010000000..2010099999} ; do echo sip:$DN@$home_domain ; done > users.csv     

>2010000000..2010099999範圍為測試帳號的數量，該指令為100000個測試帳號  


2.步驟1將產生一users.csv的檔案，將其丟入 
```sh
/usr/share/clearwater/crest/src/metaswitch/crest/tools/
``` 
3.切換至 */usr/share/clearwater/crest/src/metaswitch/crest/tools/*  
4.修改bulk_autocomplete.py檔, 把這行註解掉row.append(utils.create_secure_human_readable_id(48))並加上row.append("7kkzTyGW")在這行下面  
5.執行bulk_create.py users.csv  
```sh
/usr/share/clearwater/crest/src/metaswitch/crest/tools/bulk_create.py users.csv   
```  
>若需要啟用Application的服務，在
_initial_filter_xml = ifcs.generate_ifcs(utils.sip_uri_to_domain(public_id))_此行下面加入
_initial_filter_xml = '(ifc format)'_

>可以參考我寫的as_bulk_create.py檔 我只對接電話端(數字為奇數的帳號)設iFC

6.上一步驟將產生  
users.create_homestead_cache.casscli  
users.create_homestead_provisioning.casscli  
users.create_homestead.sh  
users.create_xdm.cqlsh  
users.create_xdm.sh  
五個檔案，將 *users.create_xdm.cqlsh、users.create_xdm.sh*移至 homer的host  
*users.create_homestead_cache.casscli 、users.create_homestead_provisioning.casscli、users.create_homestead.sh*移至 homestead的host  
7.在homestead執行   
```sh
sh users.create_homestead.sh
``` 
*切換至 homer host*  
8.執行 (執行時沒有通知，等回到可以下shell指令就代表完成)  
```sh
sh users.create_xdm.sh
```
完成以上步驟，測試帳號即建立完成。  


##STEP3:

*切換至 SIPp host*  
以下為Sipp的使用流程:

取得腳本:
Clearwater預設腳本在 /usr/share/clearwater/sip-stress/sip-stress.xml  
/usr/share/clearwater/sip-stress/ 裡面有users.csv.1檔 要注意裡面的realm是否跟存在homestead的realm一樣  
sip-stress.xml中有出現pause這個語法的要改小(單位是毫秒) 不然會暫停太久  

pause milliseconds="10000"    
pause distribution="uniform" min="0" max="600000"    

1.	可以自行撰寫腳本或者修改既有腳本

若要了解腳本內參數內容以及如何撰寫請參考中文的sipp官網文件(下列網址)  
http://sipp.sourceforge.net/doc/cn-reference.pdf  
或是參考我寫的腳本(sip-stress.xml)  

2.	切至super user，否則權限不夠發送封包  

3.	執行SIPp:  
nice -n-20 /usr/share/clearwater/bin/sipp -i (SIPp's address) -sf ./sip-stress.xml (Bono's address):5060 -t tn -s clearwater -inf ./users.csv.1 -users 2000 -m 2000 -default_behaviors all,-bye -max_socket 65000 -max_reconnect -1 -reconnect_sleep 0 -reconnect_close 0 -send_timeout 4000 -recv_timeout 12000

-i 後面的參數表示 自身的IP  
-sf 後面的參數為腳本的位置  
藍字為要發送過去封包的P-CSCF的IP以及port  
-t 後面參數表示要以tcp還是udp的方式傳送 tn為tcp、un為udp  
-s 後面的參數表示ims系統的domain (realm)   
-users後面的參數表示要會註冊多少用戶  
-m 後面表示發送幾個封包，達到這數量就會停止  
有更詳細的參數可以參考此網址  
http://sipp.sourceforge.net/doc/cn-reference.pdf  
有可以設定速率等等的參數。  

5.	觀看結果

![sipp](http://i.imgur.com/dJHrRRb.jpg)  


>產生的測試帳號一段時間後會被homestead自動清除，  
>如果發出指令後出現所有註冊都是404 not found的回應，  
>那就必須重新建立測試帳號(從STEP2的步驟5開始)  


###若是途中有重啟sprout 壓力測試掉的封包很多是正常的 需要測個5次以上才會準  
###sprout需要暖機
