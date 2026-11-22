# Stellex Loader

Stellex 鏄熀浜?NeoForge 21.1.251 婧愮爜鐨勫姞杞藉櫒鍒嗘敮锛氫繚鐣欏叏閮ㄥ寘鍚嶄笌浜岃繘鍒舵帴鍙ｏ紝鍦ㄥ唴鏍稿眰鍔犲叆 CPU 璋冨害鍣ㄣ€佸唴瀛樻不鐞嗕笌瀹㈡埛绔覆鏌撳墧闄ゃ€傚畨瑁呭櫒涓庡彂琛屽寘閮藉湪鏈粨搴撳唴锛屽鎴风涓嬭浇涓嶅啀渚濊禆瀹樻柟 NeoForge 浠撳簱銆?
## 涓嬭浇涓庡畨瑁?
鏈嶅姟绔彂琛屽寘锛歚download/stellex-loader-1.0.0-server.zip`锛圙it LFS 瀛樺偍锛夈€?
涓ょ瀹夎鏂瑰紡锛?
1. 浣跨敤瀹夎鍣細`java -jar download/stellex-loader-installer.jar --installServer <鐩綍>`銆傚畨瑁呭櫒鍐呭祵鏈粨搴撴瀯寤虹殑 universal 鍖呫€?2. 浣跨敤涓嬭浇鍣細杩愯 `download/stellex-installer.jar`锛屾垨 `java -jar download/stellex-installer.jar --install <鐩綍>`銆備笅杞藉櫒浠庢湰浠撳簱鎷夊彇鏈嶅姟绔寘骞惰В鍘嬨€?
瀹夎瀹屾垚鍚庡湪鐩綍鍐呭啓鍏?`eula.txt`锛岀敤 JDK 21 鍚姩 `run.bat`锛圵indows锛夋垨 `run.sh`銆?
## 鍐呮牳鍔熻兘

CPU 璋冨害鍣細`-Dstellex.cpu.target=60`銆乣-Dstellex.cpu.threads=6`銆乣-Dstellex.tps.guard.ms=45`銆乣-Dstellex.pregen.radius=32`銆傚懡浠?`/stellex status`銆乣/stellex target <1-100>`銆乣/stellex radius <4-256>`銆乣/stellex pause`銆乣/stellex resume`銆?
鍐呭瓨娌荤悊锛歚-Dstellex.memory.soft.percent=75`銆乣-Dstellex.memory.critical.percent=90`銆乣-Dstellex.memory.dump.on.critical=false`銆傚懡浠?`/stellex memory`銆乣/stellex memory clean`銆乣/stellex memory dump`銆?
瀹㈡埛绔墧闄わ細`config/stellex-client.properties`锛岄粯璁ゅ疄浣?128 鏍笺€佹柟鍧楀疄浣?96 鏍笺€佺粺璁￠棿闅?1200 tick銆?
## 鍏煎鎬?
- 绾?NeoForge 21.1.251 瀹㈡埛绔彲浠ヨ繘鍏?Stellex 鏈嶅姟绔€?- Stellex 瀹㈡埛绔彲浠ュ姞杞?NeoForge 妯＄粍锛涙ā缁勭殑蹇呴渶閫氶亾浠嶆寜 NeoForge 瑙勫垯鏍￠獙銆?- 鍔犺浇鍣ㄦ病鏈夋敞鍐屼换浣曞繀闇€鐨勮嚜瀹氫箟缃戠粶閫氶亾銆?
## 鏂囨。

璁烘枃瑙?`docs/PAPER.md` 涓?`docs/PAPER.docx`锛屽寘鍚瀯寤烘祦绋嬨€佸唴鏍稿疄鐜般€佸疄楠屾暟鎹笌鍏煎鎬ч獙璇併€?
## 璁稿彲

NeoForge 婧愮爜閬靛惊 LGPL-2.1锛涙湰浠撳簱鐨勪慨鏀逛笌鏂板閮ㄥ垎娌跨敤鍚屼竴璁稿彲璇併€備綔鑰咃細Mornfrost Office銆?