# 目的
ネットワークストレージ（NAS）を利用し、PCの定時バックアップを構築するものになります。

# 前提
・NAS本体及びHDDは実装、初期設定済みである

・今回は可用性を考慮し、RAID5で構築

また、下記の環境設定は必ず行うこと。

# 基本構築

1．メインPCへ静的IP割り当て

2．NASへ静的IP割り当て

3．NASにてSMBサービスを有効にする

## セキュア環境構築（NAS）

1．ホワイトリスト方式を設定し、メインPCの静的IPをホワイトリスト登録（メインPC以外接続拒否）

2．内部FWにてStaticルールを設定。メインPCの静的IPのみTCP/UDP、全ポート有効。

3．HTTPS接続有効化、HTTP接続リダイレクト設定有効化、HTTP及びHTTPSポートを任意のポートへ変更

# バッチファイル
```Batch
@echo off

: Set Local宣言
@setlocal enabledelayedexpansion

: -------------処理時間計算-------------
call :GetStartTime

:Main
: -------------初期設定-------------
: カレントフォルダ指定
set crdir=<カレントフォルダの絶対パスを指定>

: 同期までの日数
set sync=<同期までの日数を指定>

: 各ネットワークドライブの設定
: NAS IP
: set nasip=192.168.254.254など
set nasip=<NASのIPアドレスを指定>

: NAS Directory
: set nasdir=\hogeなど
set nasdir=<ネットワークドライブとして追加するNASのディレクトリ>

: NAS Account
: set nasid=hoge
: set naspw=hogehogeなど
set nasid=<NASの管理者アカウントを指定>
set naspw=<NASの管理者パスワードを指定>

: NAS Drive Symbol
: set nasadmin=Q:など
set nasadmin=<PC上に追加するネットワークドライブシンボルを指定>

: robocopy設定
set rcRetry=<同期失敗時の再試行回数>
set rcWait=<同期失敗時の待機時間>
set rcSrcPath=<同期元フォルダの絶対パス>
set rcDestPath=<同期先フォルダの絶対パス>

: -------------同期時期検出-------------
: フォルダ検出（tmpフォルダとlogフォルダがなければ作成）
If not exist %crdir%\tmp\ mkdir %crdir%\tmp\
If not exist %crdir%\log\ mkdir %crdir%\log\

: 同期更新ファイル取得
set SYNPath=%crdir%\tmp\
set SYNFile=synupdate

: タイムスタンプ用コマンド
set dirComd="DIR %SYNPath%%SYNFile% | findstr %SYNFile%"

: 同期更新ファイルタイムスタンプ
for /f "tokens=1,2" %%a in ('%dirComd%') do (
	set timeStamp=%%a %%b
)

: 現在の日時を数値化
set YYYY=%Date:~0,4%
set MM=%Date:~5,2%
set DD=%Date:~8,2%
set TT=%Time:~0,2%%Time:~3,2%
set TT=%TT: =0%
set TS=%YYYY%%MM%%DD%
: set TS=%YYYY%%MM%%DD%%TT%

: 終了の日時を数値化
set ENDYYYY=%timeStamp:~0,4%
set ENDMM=%timeStamp:~5,2%
set ENDDD=%timeStamp:~8,2%
set ENDTT=%timeStamp:~11,2%%timeStamp:~14,2%
set ENDTS=%ENDYYYY%%ENDMM%%ENDDD%
: set ENDTS=%ENDYYYY%%ENDMM%%ENDDD%%ENDTT%

: 日付差分
set /a verifyDate=%TS%-%ENDTS%

: Log出力
(
echo.
echo Value Date : %date% %time%
echo -------------- Config --------------
echo Current Folder : %crdir%
echo Sync Period : %sync% Days
echo NAS IP : %nasip%
echo NAS Drive Symbol : %nasadmin%
echo.
) >> %crdir%\log\NetworkWorkingLog_%TS%.log
(
echo -------------- Synchronize --------------
echo Sync Path : %SYNPath%
echo Sync File : %SYNFile%
echo Comp Source Date : %TS%
echo Comp Destination Date : %ENDTS%
echo Comp Period : %verifyDate%
echo.
) >> %crdir%\log\NetworkWorkingLog_%TS%.log

: --------------メインルーチン--------------
:NAS
(
echo -------------- MainRoutine --------------
echo /* NAS */
) >> %crdir%\log\NetworkWorkingLog_%TS%.log

: NAS Ping実行（Ping疎通がなければドライブ削除、疎通があればドライブ追加）
ping -n 1 -w 50 %nasip% | findstr "TTL" >> %crdir%\log\NetworkWorkingLog_%TS%.log

: NAS ネットワークドライブ処理
if %ERRORLEVEL%==0 GOTO NASADMIN
if %ERRORLEVEL%==1 GOTO NASADMINDEL

:NASADMIN
net use %nasadmin% \\%nasip%nasdir% %naspw% /user:%nasid% >> %crdir%\log\NetworkWorkingLog_%TS%.log

: 同期実行分岐
if %verifyDate% geq %sync% (
    robocopy %rcSrcPath% %rcDestPath% /mir /r:%rcRetry% /w:%rcWait% /fft /np >> %crdir%\log\NetworkWorkingLog_%TS%.log
    : 同期更新ファイル出力
    type nul > %crdir%\tmp\synupdate >> %crdir%\log\NetworkWorkingLog_%TS%.log
)
goto BATEXIT

:NASADMINDEL
net use %nasadmin% /delete >> %crdir%\log\NetworkWorkingLog_%TS%.log
goto BATEXIT

:BATEXIT
: バッチ終了処理、結果ログ出力
call :GetEndTime
(
echo.
echo -------------- Result --------------
echo Start Time : %T%
echo End Time : %T1%
echo Processing Time : %H2%h %M2%m %S2%.%C2%s
echo ------------------------------------
) >> %crdir%\log\NetworkWorkingLog_%TS%.log
endlocal
exit

:GetStartTime
: 開始時刻の取得
set T=%TIME: =0%
set H=%T:~0,2%
set M=%T:~3,2%
set S=%T:~6,2%
set C=%T:~9,2%
set /a H=1%H%-100,M=1%M%-100,S=1%S%-100,C=1%C%-100

:GetEndTime
: 終了時刻の取得
set T1=%TIME: =0%
set H1=%T1:~0,2%
set M1=%T1:~3,2%
set S1=%T1:~6,2%
set C1=%T1:~9,2%
set /a H1=1%H1%-100,M1=1%M1%-100,S1=1%S1%-100,C1=1%C1%-100

: 処理時間の計算
set /a H2=H1-H,M2=M1-M
if %M2% LSS 0 set /a H2=H2-1,M2=M2+60
 set /a S2=S1-S
if %S2% LSS 0 set /a M2=M2-1,S2=S2+60
set /a C2=C1-C
if %C2% LSS 0 set /a S2=S2-1,C2=C2+100
if %C2% LSS 10 set C2=0%C2%
```

# フローチャート
![](https://github.com/konatofu/NASDetectedTask/blob/fb8af753b362413c4a37a0da71133da132cf9f9d/image/flow.png)

# 動作説明 
## 何故ネットワークドライブを検出して処理するのか？
Windowsの特性上、ネットワークドライブへの疎通が取れず切断されている場合、何らかのアクセス（バックグラウンドアクセス含む）があった場合、疎通を取ろうとし、対象までのリーチャビリティが無いと判断するまで、同処理をリトライしようとします。

その際、OSが応答なしと判断するまでエクスプローラ上で一時的にスタックし、ユーザビリティが低下します。

故に、本バッチ実行時に対象への疎通を確認し、疎通NGであった場合は、ネットワークドライブを削除するといった手法を取っています。

# 同期処理の手法について
目的の達成に必要な手法は数多ありますが、今回はWindows標準装備である**robocopy**を採用しています。

本コマンドは、Windows7以降搭載されているコマンドである為、Windows7より前のOSでは動作しません。

# 使用方法
1．本バッチファイル内の**初期設定**部分を、自環境に合うよう書き換えて保存してください。

2．タスクスケジュールへ新規タスクを追加し、**動作トリガーをログイン時にし、本バッチファイルを実行ファイルとして登録**してください。

# Tips
バッチファイル動作時のコマンドプロンプトウィンドウが、一瞬表示されるのが嫌な方は以下を試してください。

```VB
Set sh = CreateObject("WScript.Shell")
sh.Run "<バッチファイルの絶対パス>",0,False
```

テキストファイルに上記コードを貼り付けて、拡張子(.vbs)にて保存してください。

保存したこのスクリプトを、バッチファイルの代わりにタスクスケジュールに登録してください。

# 注意事項
本バッチはrobocopyコマンドを利用しています。

`rcSrcPath`と`rcDestPath`へ入力する際は、**同期元**と**同期先**を絶対に間違わないよう注意してください。

### **同期元と同期先のフォルダパスを間違えると、同期元のデータが全て消えてしまいます。**

# 免責事項
本バッチを利用して生じた、如何なる不都合、不具合に関しては一切責任を負いません。

本バッチを実行する際は、しっかり確認した上で実行してください。
