Before playing
---
ES4ではモジュールをロードする形でアプリケーションを使用できる。
`module avail`で利用可能なモジュール一覧、`module list`で現在ロードしているモジュール一覧が見れる。
以下はGrADSとIntel fortran（ifort）をロードする際の例。
~~~
module load GrADS2/2.2.1
module load IntelMPI/2021.1.1
~~~
.bash_profile や .cshrc に記述しておくと良い。

Tutorial of running AFES
---
`とりあえず気候値の境界条件で走らせるコントロール実験（cntl）のみ記述しています。`
`境界条件を変えれば摂動実験（prtb）になりますので、わかる人は進めていただいても構いません。`

- Prepare sigle AFES
作業ディレクトリ（AFEStutorial）を作成。
~~~
cd /S/data01/G5020/`basename ${HOME}`/
cp /S/data01/G5020/c0369/AFEStutorial.tar ./
tar xvf AFEStutorial.tar
~~~

- Directory structure
基本的に <font color="DeepPink">make_nqsfiles</font>, <font color="DeepPink">run</font>, <font color="DeepPink">submit_jobs</font> 以外のディレクトリは変更しない。
You do nothing in all directories except for <font color="DeepPink">make_nqsfiles</font>, <font color="DeepPink">run</font>, and <font color="DeepPink">submit_jobs</font> in usual.
```
AFEStutorial
|-- Readme
|-- afes
|-- afes_tool
|-- bin
|-- const_data
|-- make_nqsfiles
|-- post_tools
|-- pre_tools
|-- run
`-- submit_jobs
```

- Run AFES time-slice 
基本的には以下のファイルの実験設定（特に境界条件ファイル名）を書き換えるだけ。 
```
make_nqsfiles/
|-- calendar.sh -> ../bin/calendar.sh
|-- mknqs_cntl.sh
`-- mknqs_prtb.sh
```
cntl実験のNQSファイル（スパコンへの命令書）を作る。
```
cd make_nqsfiles/
./mknqs_cntl.sh 5
```
mknqs\_cntl.sh中の変数で実験設定に関わるもの
　<font color="DeepPink">casename</font>：任意の実験名（arbitrary name of run）
　<font color="DeepPink">bsst</font>：SSTのファイル名（SST's file name）
　<font color="DeepPink">bice</font>：SICのファイル名（SIC's file name）
　※上記の境界条件ファイルを変更することで別実験（例、摂動実験=prtb）を行う。
　※SST, SICの境界条件ファイルについては後で記述（preparing boundary condition is noted later)
　Tres Lres kmax jmax：水平解像度（境界条件ファイルなど用意する必要あり。）
　num_mpiprocs num_nodes ve_pnode ntask：並列実行に関するもの
　elapse：実実行時間（T79L56だと1年積分に2時間程度。上限は48hr。）
　intg_month：積分期間（月）
　チュートリアルでは、48時間以内に終わる15年積分（180か月）[1job]を5回（mknqs_cntl.shの引数）続けて投入することで75年積分を行っている。
jobを投入（`qsub`）する。
```
cd ../submit_jobs/cntl
qsub cntl1979.nqs cntl1994.nqs cntl2009.nqs cntl.2024.nqs cntl2039.nqs
```
`qstat`, `reqstat`で状況確認ができる（経過時間など）。
途中で止めたい場合は`qdel RequestID`でjobを止めることができる。Request IDはqstatの一列目の数字。

- Post process計算が終わると AFEStutorial/run に cntl というディレクトリができる。
```
run
|-- BC
|-- IC
`-- cntl
    |-- IC
    |-- log
    `-- out
        |-- 1994010100_end
        |-- 2009010100_end
        |-- 2024010100_end
        |-- 2039010100_end
        |-- 2054010100_end
        |-- 2069010100_end
        |-- 2084010100_end
        `-- 2nd_product
            |-- cntl
            `-- cntl.MNMEAN.ctl
```
run直下の BC IC は外部条件ファイルの格納場所、cntl/IC には restart file が格納される。
cntl/out/yyyymmddhh_end にはモデル出力（日平均データ）、
cntl/out/2nd_product には二次加工されたデータ（月平均データ）が格納される。
基本的に多年数積分するタイムスライス実験の場合は二次プロダクトの月平均データだけ見れば良い。
二次プロダクトの生成は AFEStutorial/post_tools のプログラムを参照。

- Notes for computation resources`rusg`というコマンドでリソース使用状況がわかる。
グループのリソース利用上限（計算時間とデータ領域使用量）を超えると、`全員のjobが止まってしまう`ので注意。
※特にデータ領域使用量Prepare condition files---
SSTとSICの1981−2010年気候値を作り、境界条件ファイルにする。- AFEStutorial/run/BC/Merged 内で、Merged Hadley/OISST dataset から1981−2010の気候値データを作り、AFES形式に変換する。
```
BC/Merged/Merged.org/
|-- clim
|   |-- MODEL.ICE.Clim.81-10.bin
|   |-- MODEL.ICE.Clim.81-10.ctl_d16
|   |-- MODEL.SST.Clim.81-10.bin
|   |-- MODEL.SST.Clim.81-10.ctl_d16
|   `-- make_ctlfiles.csh_d16
`-- orgdata
    |-- MODEL.ICE.HAD187001-198110.OI198111-201903.nc
    |-- MODEL.SST.HAD187001-198110.OI198111-201903.nc
    `-- make_clim.gs
```

- オリジナルのnetcdf形式ファイルから気候値（Grads形式）を作る。
```
cd BC/Merged/Merged.org/
grads -blc make_clim.gs
cd ../clim/
./make_ctlfiles.csh_d16
ls -la
```

- Grads形式の気候値ファイルをAFES形式に変換し(afes.P1)、並列数24(afes.P24)に分割する。
~~~
cd ../../
./makebc_grads.csh
tree -P "T79*Clim.81-10*"
.
|-- Merged.d2PDF
|-- Merged.org
|-- afes.P1
|   |-- T79P1_ICE.Merd16.Clim.81-10.p_0000
|   `-- T79P1_SST.Merd16.Clim.81-10.p_0000
`-- afes.P24
    |-- T79P24_ICE.Merd16.Clim.81-10.p_[0000-0023]
    `-- T79P24_SST.Merd16.Clim.81-10.p_[0000-0023]
~~~

- AFES形式ファイルについて
AFES形式のファイルの構造は fortran 風に言えば
~~~
character :: header(64)*16
real*8,allocatable :: var8(:,:,:)
~~~
がいくつも繋がった sequential file である（big_endian）。
※具体的には AFEStutorial/run/BC/Merged/read_gt3files.f90 を参照。
header(3)：変数名
header(27)：時刻
は実際にAFESで読み込まれる部分のため特に重要。
その他のheader部分は実際にはAFESでは読み込まれない（と思う）が、昔からの日本のAGCMのformatを踏襲している。