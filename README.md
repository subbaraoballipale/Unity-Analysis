# IL-Practice
Unityで作られたapkファイルを解析してみる

## apkをデコンパイルする
apkはダウンローダサイトからダウンロードしたり, Androidにインストールしたデータから構築したり, 様々な方法で入手出来る  
apkのデコンパイルに関しては, apktoolを用いる  
http://ibotpeaches.github.io/Apktool/

apktoolで以下のように実行するとデコンパイルすることが出来る
```sh
$ apktool d apkの名前
```

## Assembly-CSharp.dllを読む
デコンパイルすると, ```assets```->```bin```->```Data```->```Managed```と辿って行くと, ```Assembly-CSharp.dll```がある  
ここにUnity側で書かれたC#のコードが書かれているので, ILSpyやdnSpy, dotPeekなどを用いてデコンパイルする  

ILSpyなどを用いてデコンパイルをした場合, 元のようにDLLを再コンパイルするのは難しいため, ildasmを使う  
ildasmでDLLをILに変換し, ILSpyなどでデコンパイルしたC#のコードを元にILを書き換え, ilasmで再コンパイルする  

## apkを作成し, デバッグする
DLLを再コンパイルしたら, 元のディレクトリにDLLを移し, apktoolでapkを再コンパイルする  
このとき```apktool.yml```に```doNotCompress:```があった場合は, それ以下の行を全て消して上書き保存しないとビルドが通らない

```sh
$ apktool b フォルダ名 -o apkの名前
```

apkを再コンパイルしただけではAndroid側でインストール出来ないため, 署名を行う  
オレオレの証明書を作成し, 署名するには以下のように```keytool```と```jarsigner```を使う
```sh
# 署名を作成する
$ keytool -genkey -dname "c=JP" -keypass パスワード -keystore keystoreの名前 -storepass パスワード -validity 10000 -alias エイリアス -keyalg RSA
# 署名する
$ jarsigner -digestalg SHA1 -verbose -signedjar apkの名前 -keystore keystoreの名前 apkの名前 エイリアス -sigalg MD5withRSA -digestalg SHA1
```

実機にインストールする場合はapkをAndroidにコピーしてから行っても良いし, adbを用いても良い
```sh
$ adb install apkの名前
```

## 処理を書き換える
今回はゲーム中でどのような状況でも必ずPerfect判定が出るように, 各ノートの判定部分を書き換える

<hr>

ゲーム全体を追っていくと, Alto.Scene.InGameでゲームに関するコードを発見  
更に中を追っていくと, InGameDirectorクラスのUpdate関数付近から全ての処理が動いているように見える  
- Alto.Scene.InGame  
    - InGameDirector::Update()  
        - InGameManagerBase::UpdatePlayState()
            - InGameManagerBase::ExecUpdate()  
                - InputManager::ExecInput()  
                    - InputManager::InputPlaying()  
                        - InputManager::InputButton()  
                            - ButtonBase::ExecTouchBegan()  
                            - ButtonBase::ExecTouchMoved()  
                            - ButtonBase::ExecTouchEnded()  
                                - GirlButton::ExecTouchBegan()  
                                - GirlButton::ExecTouchMoved()  
                                - GirlButton::ExecTouchEnded()  
                                    - NoteUtility::GetResult()  

タップの判定を行っていたのは, NoteUtilityクラスのGetResult関数だと思われる  
直接数値から結果を判定しているので, 返り値を書き換えてみる

以下のように変更すると
```il
  .method public hidebysig static valuetype Alto.Scene.InGame.NoteResultType 
          GetResult(float32 diffSecond) cil managed
  {
    // コード サイズ       75 (0x4b)
    .maxstack  7
    IL_0000:  ldarg.0
    IL_0001:  call       float32 [UnityEngine]UnityEngine.Mathf::Abs(float32)
    IL_0006:  starg.s    diffSecond
    IL_0008:  ldarg.0
    IL_0009:  ldc.r4     5.0000001e-002
    IL_000e:  bge.un     IL_0015

    IL_0013:  ldc.i4.4
    IL_0014:  ret

    IL_0015:  ldarg.0
    IL_0016:  ldc.r4     0.13333334
    IL_001b:  bge.un     IL_0022

    IL_0020:  ldc.i4.4
    IL_0021:  ret

    IL_0022:  ldarg.0
    IL_0023:  ldc.r4     0.25
    IL_0028:  bge.un     IL_002f

    IL_002d:  ldc.i4.4
    IL_002e:  ret

    IL_002f:  ldarg.0
    IL_0030:  ldc.r4     0.33333334
    IL_0035:  bge.un     IL_003c

    IL_003a:  ldc.i4.4
    IL_003b:  ret

    IL_003c:  ldarg.0
    IL_003d:  ldc.r4     0.41666666
    IL_0042:  bge.un     IL_0049

    IL_0047:  ldc.i4.4
    IL_0048:  ret

    IL_0049:  ldc.i4.m1
    IL_004a:  ret
  } // end of method NoteUtility::GetResult
```

上記のコードを試してみた結果, プレイ中にノートが降ってきたタイミングでタップをすると必ずPerfect判定が出るようになったが, タップしないでスルーすると通常どおりエラー判定となった  

<hr>

タップせずスルーした結果Miss判定が出ていると思われるので, NoteResultType.Missを追ってみる

- Alto.Scene.InGame.NoteLong::ExecOverWaitState()
- Alto.Scene.InGame.NoteLong::ExecOverStopState()
- Alto.Scene.InGame.NoteLong::ExecTouchBegan()
- Alto.Scene.InGame.NoteSingleBase::ExecOverMoveState()  
(今回は関係ないが, NoteResultType.Miss.InGamerecord::AddNoteCount()でスコアを計算していると推測出来る)

これらで直接的にNoteResultType.Missを扱っているため, その部分を直接NoteResultType.Perfectに変更する

```il
  .method family hidebysig newslot virtual 
          instance void  ExecOverWaitState() cil managed
  {
    // コード サイズ       107 (0x6b)
    .maxstack  36
    .locals init (int32 V_0)
    IL_0000:  ldarg.0
    IL_0001:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0006:  ldc.i4.1
    IL_0007:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_000c:  ldarg.0
    IL_000d:  call       instance void Alto.Scene.InGame.NoteFlontBase::RecordFirstEffectedActionLog()
    IL_0012:  ldarg.0
    IL_0013:  ldc.i4.0
    IL_0014:  call       instance int32 Alto.Scene.InGame.NoteFlontBase::CalcAddDamage(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0019:  stloc.0
    IL_001a:  ldarg.0
    IL_001b:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0020:  ldarg.0
    IL_0021:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0026:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_002b:  ldarg.0
    IL_002c:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0031:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_0036:  ldarg.0
    IL_0037:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_003c:  callvirt   instance valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.ButtonBase::get_ButtonType_()
    IL_0041:  ldc.i4.0
    IL_0042:  newobj     instance void [mscorlib]System.Decimal::.ctor(int32)
    IL_0047:  ldloc.0
    IL_0048:  ldarg.0
    IL_0049:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_004e:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_0053:  ldc.i4.m1
    IL_0054:  ldc.i4.1
    IL_0055:  ldc.i4.s   4
    IL_0056:  ldc.i4.s   4
    IL_0057:  ldc.i4.0
    IL_0058:  ldc.i4.0
    IL_0059:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_005e:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_0063:  ldarg.0
    IL_0064:  ldc.i4.1
    IL_0065:  callvirt   instance void Alto.Scene.InGame.NoteLong::Deactivate(bool)
    IL_006a:  ret
  } // end of method NoteLong::ExecOverWaitState
```

```il
  .method family hidebysig newslot virtual 
          instance void  ExecOverStopState() cil managed
  {
    // コード サイズ       194 (0xc2)
    .maxstack  54
    .locals init (int32 V_0,
             valuetype [mscorlib]System.Decimal V_1)
    IL_0000:  ldarg.0
    IL_0001:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0006:  ldc.i4.1
    IL_0007:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_000c:  ldarg.0
    IL_000d:  ldc.i4.0
    IL_000e:  call       instance int32 Alto.Scene.InGame.NoteFlontBase::CalcAddDamage(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0013:  stloc.0
    IL_0014:  ldarg.0
    IL_0015:  ldarg.0
    IL_0016:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_001b:  ldc.i4.3
    IL_001c:  callvirt   instance class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType> Alto.Scene.InGame.GirlButton::GetList(valuetype Alto.Utility.BlowType)
    IL_0021:  call       instance bool Alto.Scene.InGame.NoteLong::AddScoreChanceList(class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType>)
    IL_0026:  pop
    IL_0027:  ldarg.0
    IL_0028:  ldarg.0
    IL_0029:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_002e:  ldc.i4.4
    IL_002f:  callvirt   instance class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType> Alto.Scene.InGame.GirlButton::GetList(valuetype Alto.Utility.BlowType)
    IL_0034:  call       instance bool Alto.Scene.InGame.NoteLong::AddPerfectChanceList(class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType>)
    IL_0039:  pop
    IL_003a:  ldarg.0
    IL_003b:  ldfld      valuetype [mscorlib]System.Decimal Alto.Scene.InGame.NoteLong::touchBeganAddScore_
    IL_0040:  stloc.1
    IL_0041:  ldloc.1
    IL_0042:  ldarg.0
    IL_0043:  call       instance valuetype [mscorlib]System.Decimal Alto.Scene.InGame.NoteLong::GetScoreChanceRate()
    IL_0048:  call       valuetype [mscorlib]System.Decimal [mscorlib]System.Decimal::op_Multiply(valuetype [mscorlib]System.Decimal,
                                                                                                  valuetype [mscorlib]System.Decimal)
    IL_004d:  stloc.1
    IL_004e:  ldarg.0
    IL_004f:  ldarg.0
    IL_0050:  ldfld      valuetype Alto.Scene.InGame.NoteResultType Alto.Scene.InGame.NoteLong::touchBeganResultType_
    IL_0055:  call       instance bool Alto.Scene.InGame.NoteFlontBase::IsPerfectPlus(valuetype Alto.Scene.InGame.NoteResultType)
    IL_005a:  brfalse    IL_006c

    IL_005f:  ldloc.1
    IL_0060:  ldarg.0
    IL_0061:  call       instance valuetype [mscorlib]System.Decimal Alto.Scene.InGame.NoteLong::GetPerfectChanceRate()
    IL_0066:  call       valuetype [mscorlib]System.Decimal [mscorlib]System.Decimal::op_Multiply(valuetype [mscorlib]System.Decimal,
                                                                                                  valuetype [mscorlib]System.Decimal)
    IL_006b:  stloc.1
    IL_006c:  ldarg.0
    IL_006d:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0072:  ldarg.0
    IL_0073:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0078:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_007d:  ldarg.0
    IL_007e:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0083:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_0088:  ldarg.0
    IL_0089:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_008e:  ldfld      valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::buttonType_
    IL_0093:  ldloc.1
    IL_0094:  ldloc.0
    IL_0095:  ldarg.0
    IL_0096:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_009b:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_00a0:  ldc.i4.m1
    IL_00a1:  ldc.i4.1
    IL_00a2:  ldc.i4.s   4
    IL_00a3:  ldc.i4.s   4
    IL_00a4:  ldc.i4.0
    IL_00a5:  ldarg.0
    IL_00a6:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_00ab:  ldfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isEffectedMirrorBonus_
    IL_00b0:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_00b5:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_00ba:  ldarg.0
    IL_00bb:  ldc.i4.1
    IL_00bc:  callvirt   instance void Alto.Scene.InGame.NoteLong::Deactivate(bool)
    IL_00c1:  ret
  } // end of method NoteLong::ExecOverStopState
```

```il
  .method family hidebysig newslot virtual 
          instance void  ExecOverMoveState() cil managed
  {
    // コード サイズ       99 (0x63)
    .maxstack  32
    IL_0000:  ldarg.0
    IL_0001:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0006:  ldc.i4.1
    IL_0007:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_000c:  ldarg.0
    IL_000d:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0012:  ldarg.0
    IL_0013:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0018:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_001d:  ldarg.0
    IL_001e:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0023:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_0028:  ldarg.0
    IL_0029:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_002e:  ldfld      valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::buttonType_
    IL_0033:  ldc.i4.0
    IL_0034:  newobj     instance void [mscorlib]System.Decimal::.ctor(int32)
    IL_0039:  ldarg.0
    IL_003a:  ldc.i4.0
    IL_003b:  call       instance int32 Alto.Scene.InGame.NoteFlontBase::CalcAddDamage(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0040:  ldarg.0
    IL_0041:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_0046:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_004b:  ldc.i4.m1
    IL_004c:  ldc.i4.0
    IL_004d:  ldc.i4.s   4
    IL_004e:  ldc.i4.s   4
    IL_004f:  ldc.i4.0
    IL_0050:  ldc.i4.0
    IL_0051:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_0056:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_005b:  ldarg.0
    IL_005c:  ldc.i4.1
    IL_005d:  callvirt   instance void Alto.Scene.InGame.NoteFlontBase::Deactivate(bool)
    IL_0062:  ret
  } // end of method NoteSingleBase::ExecOverMoveState
```

上記のコードを試してみた結果, 判定は全てPerfectとなったが, スコアとライフが正常に減っていってしまう  
どうやらスコア計算やコンボ計算部分を書き換えないといけないようだ

<hr>

よく見てみると, 上記の関数は全てOneFrameData構造体に値を渡しているが, OneFrameData構造体は以下のようになっている
```csharp
public OneFrameData(float musicSec, int index, ButtonType buttonType, decimal addScore, int addPower, bool isProtect, int addCombo, NoteType note, NoteResultType inResult, NoteResultType outResult, bool isBlueText, bool isMirrorBonus)
{
    this.musicSec_ = musicSec;
    this.index_ = index;
    this.buttonType_ = buttonType;
    this.addScore_ = addScore;
    this.addPower_ = addPower;
    this.isProtect_ = isProtect;
    this.addCombo_ = addCombo;
    this.note_ = note;
    this.inResult_ = inResult;
    this.outResult_ = outResult;
    this.isBlueText_ = isBlueText;
    this.isMirrorBonus_ = isMirrorBonus;
}
```

このことから分かるように, addScoreやaddPower, addCombo, isMirrorBonusも書き換えることで上手く行きそうだ

実際に書き換えてみると
```il
  .method family hidebysig newslot virtual 
          instance void  ExecOverWaitState() cil managed
  {
    // コード サイズ       107 (0x6b)
    .maxstack  36
    .locals init (int32 V_0)
    IL_0000:  ldarg.0
    IL_0001:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0006:  ldc.i4.1
    IL_0007:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_000c:  ldarg.0
    IL_000d:  call       instance void Alto.Scene.InGame.NoteFlontBase::RecordFirstEffectedActionLog()
    IL_0012:  ldarg.0
    IL_0013:  ldc.i4.0
    IL_0014:  call       instance int32 Alto.Scene.InGame.NoteFlontBase::CalcAddDamage(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0019:  stloc.0
    IL_001a:  ldarg.0
    IL_001b:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0020:  ldarg.0
    IL_0021:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0026:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_002b:  ldarg.0
    IL_002c:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0031:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_0036:  ldarg.0
    IL_0037:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_003c:  callvirt   instance valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.ButtonBase::get_ButtonType_()
    IL_0041:  ldc.i4.s   1000
    IL_0042:  newobj     instance void [mscorlib]System.Decimal::.ctor(int32)
    IL_0047:  ldc.i4.s   0
    IL_0048:  ldarg.0
    IL_0049:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_004e:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_0053:  ldc.i4.1
    IL_0054:  ldc.i4.1
    IL_0055:  ldc.i4.s   4
    IL_0056:  ldc.i4.s   4
    IL_0057:  ldc.i4.s   1
    IL_0058:  ldc.i4.s   1
    IL_0059:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_005e:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_0063:  ldarg.0
    IL_0064:  ldc.i4.1
    IL_0065:  callvirt   instance void Alto.Scene.InGame.NoteLong::Deactivate(bool)
    IL_006a:  ret
  } // end of method NoteLong::ExecOverWaitState
```

```il
  .method family hidebysig newslot virtual 
          instance void  ExecOverStopState() cil managed
  {
    // コード サイズ       194 (0xc2)
    .maxstack  54
    .locals init (int32 V_0,
             valuetype [mscorlib]System.Decimal V_1)
    IL_0000:  ldarg.0
    IL_0001:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0006:  ldc.i4.1
    IL_0007:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_000c:  ldarg.0
    IL_000d:  ldc.i4.0
    IL_000e:  call       instance int32 Alto.Scene.InGame.NoteFlontBase::CalcAddDamage(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0013:  stloc.0
    IL_0014:  ldarg.0
    IL_0015:  ldarg.0
    IL_0016:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_001b:  ldc.i4.3
    IL_001c:  callvirt   instance class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType> Alto.Scene.InGame.GirlButton::GetList(valuetype Alto.Utility.BlowType)
    IL_0021:  call       instance bool Alto.Scene.InGame.NoteLong::AddScoreChanceList(class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType>)
    IL_0026:  pop
    IL_0027:  ldarg.0
    IL_0028:  ldarg.0
    IL_0029:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_002e:  ldc.i4.4
    IL_002f:  callvirt   instance class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType> Alto.Scene.InGame.GirlButton::GetList(valuetype Alto.Utility.BlowType)
    IL_0034:  call       instance bool Alto.Scene.InGame.NoteLong::AddPerfectChanceList(class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType>)
    IL_0039:  pop
    IL_003a:  nop
    IL_003b:  ldc.i4.s   1000
    IL_0040:  stloc.1
    IL_0041:  ldloc.1
    IL_0042:  nop
    IL_0043:  ldc.i4.s   1
    IL_0048:  call       valuetype [mscorlib]System.Decimal [mscorlib]System.Decimal::op_Multiply(valuetype [mscorlib]System.Decimal,
                                                                                                  valuetype [mscorlib]System.Decimal)
    IL_004d:  stloc.1
    IL_004e:  ldarg.0
    IL_004f:  ldarg.0
    IL_0050:  ldfld      valuetype Alto.Scene.InGame.NoteResultType Alto.Scene.InGame.NoteLong::touchBeganResultType_
    IL_0055:  call       instance bool Alto.Scene.InGame.NoteFlontBase::IsPerfectPlus(valuetype Alto.Scene.InGame.NoteResultType)
    IL_005a:  brfalse    IL_006c

    IL_005f:  ldloc.1
    IL_0060:  ldarg.0
    IL_0061:  call       instance valuetype [mscorlib]System.Decimal Alto.Scene.InGame.NoteLong::GetPerfectChanceRate()
    IL_0066:  call       valuetype [mscorlib]System.Decimal [mscorlib]System.Decimal::op_Multiply(valuetype [mscorlib]System.Decimal,
                                                                                                  valuetype [mscorlib]System.Decimal)
    IL_006b:  stloc.1
    IL_006c:  ldarg.0
    IL_006d:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0072:  ldarg.0
    IL_0073:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0078:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_007d:  ldarg.0
    IL_007e:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0083:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_0088:  ldarg.0
    IL_0089:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_008e:  ldfld      valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::buttonType_
    IL_0093:  ldloc.1
    IL_0094:  ldc.i4.s   0
    IL_0095:  ldarg.0
    IL_0096:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_009b:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_00a0:  ldc.i4.1
    IL_00a1:  ldc.i4.1
    IL_00a2:  ldc.i4.s   4
    IL_00a3:  ldc.i4.s   4
    IL_00a4:  ldc.i4.s   1
    IL_00a5:  ldc.i4.s   1
    IL_00a6:  nop
    IL_00ab:  nop
    IL_00b0:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_00b5:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_00ba:  ldarg.0
    IL_00bb:  ldc.i4.1
    IL_00bc:  callvirt   instance void Alto.Scene.InGame.NoteLong::Deactivate(bool)
    IL_00c1:  ret
  } // end of method NoteLong::ExecOverStopState
```

```il
  .method public hidebysig virtual instance void 
          ExecTouchBegan(valuetype [UnityEngine]UnityEngine.Vector2 touchPos,
                         valuetype Alto.Scene.InGame.NoteResultType result) cil managed
  {
    // コード サイズ       356 (0x164)
    .maxstack  83
    .locals init (valuetype Alto.Scene.InGame.NoteResultType V_0,
             int32 V_1,
             valuetype [mscorlib]System.Decimal V_2,
             valuetype [mscorlib]System.Decimal V_3)
    IL_0000:  ldarg.2
    IL_0001:  ldc.i4.m1
    IL_0002:  bne.un     IL_0008

    IL_0007:  ret

    IL_0008:  ldarg.0
    IL_0009:  call       instance void Alto.Scene.InGame.NoteFlontBase::RecordFirstEffectedActionLog()
    IL_000e:  ldarg.0
    IL_000f:  ldarg.2
    IL_0010:  call       instance valuetype Alto.Scene.InGame.NoteResultType Alto.Scene.InGame.NoteFlontBase::GetNoteResultType(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0015:  stloc.0
    IL_0016:  ldarg.0
    IL_0017:  ldloc.0
    IL_0018:  stfld      valuetype Alto.Scene.InGame.NoteResultType Alto.Scene.InGame.NoteLong::touchBeganResultType_
    IL_001d:  ldarg.0
    IL_001e:  ldloc.0
    IL_001f:  call       instance int32 Alto.Scene.InGame.NoteFlontBase::CalcAddDamage(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0024:  stloc.1
    IL_0025:  ldarg.0
    IL_0026:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_002b:  ldloc.0
    IL_002c:  callvirt   instance void Alto.Scene.InGame.GirlButton::PlayTapAnimation(valuetype Alto.Scene.InGame.NoteResultType)
    IL_0031:  ldarg.2
    IL_0032:  brtrue     IL_009e

    IL_0037:  ldarg.0
    IL_0038:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_003d:  ldc.i4.1
    IL_003e:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_0043:  ldarg.0
    IL_0044:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0049:  ldarg.0
    IL_004a:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_004f:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_0054:  ldarg.0
    IL_0055:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_005a:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_005f:  ldarg.0
    IL_0060:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0065:  ldfld      valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::buttonType_
    IL_006a:  ldc.i4     1000
    IL_006b:  newobj     instance void [mscorlib]System.Decimal::.ctor(int32)
    IL_0070:  ldc.i4.s   0
    IL_0071:  ldarg.0
    IL_0072:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_0077:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_007c:  ldc.i4.1
    IL_007d:  ldc.i4.1
    IL_007e:  ldc.i4.s   4
    IL_007f:  ldc.i4.s   4
    IL_0080:  ldc.i4.s   1
    IL_0081:  ldc.i4.s   1
    IL_0082:  nop
    IL_0087:  nop
    IL_008c:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_0091:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_0096:  ldarg.0
    IL_0097:  ldc.i4.1
    IL_0098:  callvirt   instance void Alto.Scene.InGame.NoteLong::Deactivate(bool)
    IL_009d:  ret

    IL_009e:  ldarg.0
    IL_009f:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_00a4:  callvirt   instance class Alto.Scene.InGame.InGameScoreLogicData Alto.Scene.InGame.InGameManagerBase::get_InGameScoreLogicData_()
    IL_00a9:  ldarg.0
    IL_00aa:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_00af:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsSameAttribute()
    IL_00b4:  callvirt   instance valuetype [mscorlib]System.Decimal Alto.Scene.InGame.InGameScoreLogicData::GetFixScore(bool)
    IL_00b9:  stloc.2
    IL_00ba:  ldarg.0
    IL_00bb:  ldloc.2
    IL_00bc:  ldloc.0
    IL_00bd:  call       instance valuetype [mscorlib]System.Decimal Alto.Scene.InGame.NoteFlontBase::CalcBaseCorrentedScore(valuetype [mscorlib]System.Decimal,
                                                                                                                             valuetype Alto.Scene.InGame.NoteResultType)
    IL_00c2:  stloc.3
    IL_00c3:  ldarg.0
    IL_00c4:  ldloc.3
    IL_00c5:  stfld      valuetype [mscorlib]System.Decimal Alto.Scene.InGame.NoteLong::touchBeganAddScore_
    IL_00ca:  ldarg.0
    IL_00cb:  ldloc.0
    IL_00cc:  call       instance bool Alto.Scene.InGame.NoteFlontBase::IsPerfectPlus(valuetype Alto.Scene.InGame.NoteResultType)
    IL_00d1:  brfalse    IL_00d8

    IL_00d6:  ldc.i4.5
    IL_00d7:  stloc.0
    IL_00d8:  ldarg.0
    IL_00d9:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_00de:  ldarg.0
    IL_00df:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_00e4:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_00e9:  ldarg.0
    IL_00ea:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_00ef:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_00f4:  ldarg.0
    IL_00f5:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_00fa:  ldfld      valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::buttonType_
    IL_00ff:  ldc.i4.0
    IL_0100:  newobj     instance void [mscorlib]System.Decimal::.ctor(int32)
    IL_0105:  ldloc.1
    IL_0106:  ldarg.0
    IL_0107:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_010c:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_0111:  ldc.i4.0
    IL_0112:  ldc.i4.3
    IL_0113:  ldc.i4.m1
    IL_0114:  ldloc.0
    IL_0115:  ldc.i4.0
    IL_0116:  ldarg.0
    IL_0117:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_011c:  ldfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isEffectedMirrorBonus_
    IL_0121:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_0126:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_012b:  ldarg.0
    IL_012c:  ldarg.0
    IL_012d:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_0132:  ldc.i4.3
    IL_0133:  callvirt   instance class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType> Alto.Scene.InGame.GirlButton::GetList(valuetype Alto.Utility.BlowType)
    IL_0138:  call       instance bool Alto.Scene.InGame.NoteLong::AddScoreChanceList(class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType>)
    IL_013d:  pop
    IL_013e:  ldarg.0
    IL_013f:  ldarg.0
    IL_0140:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_0145:  ldc.i4.4
    IL_0146:  callvirt   instance class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType> Alto.Scene.InGame.GirlButton::GetList(valuetype Alto.Utility.BlowType)
    IL_014b:  call       instance bool Alto.Scene.InGame.NoteLong::AddPerfectChanceList(class [mscorlib]System.Collections.Generic.List`1<valuetype Alto.Scene.InGame.ButtonType>)
    IL_0150:  pop
    IL_0151:  ldarg.0
    IL_0152:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_0157:  callvirt   instance void Alto.Scene.InGame.GirlButton::PlayLongTapParticle()
    IL_015c:  ldarg.0
    IL_015d:  ldc.i4.2
    IL_015e:  call       instance void Alto.Scene.InGame.NoteBase::set_State_(valuetype Alto.Scene.InGame.NoteState)
    IL_0163:  ret
  } // end of method NoteLong::ExecTouchBegan
```

```il
.method family hidebysig newslot virtual 
          instance void  ExecOverMoveState() cil managed
  {
    // コード サイズ       99 (0x63)
    .maxstack  32
    IL_0000:  ldarg.0
    IL_0001:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0006:  ldc.i4.1
    IL_0007:  stfld      bool Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::isResult_
    IL_000c:  ldarg.0
    IL_000d:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0012:  ldarg.0
    IL_0013:  call       instance class Alto.Scene.InGame.InGameManagerBase Alto.Scene.InGame.NoteBase::get_InGameManager_()
    IL_0018:  callvirt   instance float32 Alto.Scene.InGame.InGameManagerBase::GetSoundSec()
    IL_001d:  ldarg.0
    IL_001e:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_0023:  ldfld      int32 Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::index_
    IL_0028:  ldarg.0
    IL_0029:  call       instance class Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation Alto.Scene.InGame.NoteFlontBase::get_InfoData_()
    IL_002e:  ldfld      valuetype Alto.Scene.InGame.ButtonType Alto.Scene.InGame.NoteBatchInfomation/NoteInfomation::buttonType_
    IL_0033:  ldc.i4.s   1000
    IL_0034:  newobj     instance void [mscorlib]System.Decimal::.ctor(int32)
    IL_0039:  nop
    IL_003a:  ldc.i4.s   0
    IL_003b:  nop
    IL_0040:  ldarg.0
    IL_0041:  call       instance class Alto.Scene.InGame.GirlButton Alto.Scene.InGame.NoteBase::get_TargetButton_()
    IL_0046:  callvirt   instance bool Alto.Scene.InGame.GirlButton::IsEffectedProtect()
    IL_004b:  ldc.i4.1
    IL_004c:  ldc.i4.0
    IL_004d:  ldc.i4.s   4
    IL_004e:  ldc.i4.s   4
    IL_004f:  ldc.i4.s   1
    IL_0050:  ldc.i4.s   1
    IL_0051:  newobj     instance void Alto.Scene.InGame.OneFrameData::.ctor(float32,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.ButtonType,
                                                                             valuetype [mscorlib]System.Decimal,
                                                                             int32,
                                                                             bool,
                                                                             int32,
                                                                             valuetype Alto.Scene.InGame.NoteType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             valuetype Alto.Scene.InGame.NoteResultType,
                                                                             bool,
                                                                             bool)
    IL_0056:  callvirt   instance void Alto.Scene.InGame.InGameManagerBase::RegisterOneFrameGameData(valuetype Alto.Scene.InGame.OneFrameData)
    IL_005b:  ldarg.0
    IL_005c:  ldc.i4.1
    IL_005d:  callvirt   instance void Alto.Scene.InGame.NoteFlontBase::Deactivate(bool)
    IL_0062:  ret
  } // end of method NoteSingleBase::ExecOverMoveState
```

これで全てのノートがPerfect判定で, かつコンボ等もフルで処理されるようになる
