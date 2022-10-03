<!--
  WAR ファイルをビルドして WebLogic Server 12c にデプロイする Ant スクリプト
  
  - [WebLogic Server に WAR ファイルをデプロイする Ant スクリプト](https://neos21.net/blog/2017/01/02-02.html)
  - [WebLogic Server に WAR ファイルをデプロイする Ant スクリプトの改善版](https://neos21.net/blog/2017/01/03-02.html)
-->
<?xml version="1.0" encoding="UTF-8"?>
<project name="TestProject" default="Main" basedir=".">
  
  <!--
    - このファイルがプロジェクトディレクトリの直下に置いてあること
    - project 要素の name 属性値はビルドやリフレッシュを行う対象のプロジェクト名にしておくこと
    - 「外部ツールの構成」→「JRE」タブ→「ランタイム JRE」→「ワークスペースと同じ JRE で実行」を設定しておくこと
  -->
  
  <!-- プロジェクトのディレクトリを絶対パスで指定する (このファイルがプロジェクトディレクトリの直下に置いてあること) -->
  <dirname property="base" file="${ant.file}"/>
  
  <!-- プロジェクト名 (project 要素の name 属性値を使用している・ビルドやリフレッシュを行う対象のプロジェクト名にしておくこと) -->
  <property name="projectName" value="${ant.project.name}"/>
  
  <!-- 生成する WAR ファイル名 -->
  <property name="warName" value="${projectName}"/>
  
  <!-- WAR ファイルのパス : プロジェクトの親ディレクトリにしておく -->
  <property name="warPath" value="${base}/../${warName}.war"/>
  
  <!-- WLS アップロード先のフルパス -->
  <property name="uploadPath" value="C:/Oracle/Middleware/Oracle_Home/user_projects/domains/base_domain/servers/AdminServer/upload/${warName}.war"/>
  
  <!-- WLS デプロイ先ホスト -->
  <property name="adminUrl" value="t3://localhost:7001"/>
  
  <!-- WLS ログインユーザ名 -->
  <property name="userName" value="Neo"/>
  
  <!-- WLS ログインパスワード -->
  <property name="password" value="SomethingPassword"/>
  
  <!-- WLS デプロイ先サーバ名 -->
  <property name="target" value="AdminServer"/>
  
  <!-- WLS アプリ名 -->
  <property name="aplName" value="${warName}"/>
  
  <!-- WLS のデプロイ用コマンドを wls 要素として定義する -->
  <taskdef name="wls" classname="weblogic.ant.taskdefs.management.WLDeploy">
    <classpath>
      <pathelement location="C:/Oracle/Middleware/Oracle_Home/wlserver/server/lib/weblogic.jar"/>
    </classpath>
  </taskdef>
  
  <target name="Main" depends="Build, Deploy" description="WAR ファイルをビルドして WLS にデプロイする">
    <echo message="[${projectName}] Finished."/>
  </target>
  
  <target name="Build" depends="Refresh, FullBuild, CreateWar" description="WAR ファイルをビルドする">
    <echo message="[${projectName}] Deploy Finished."/>
  </target>
  
  <target name="Deploy" depends="StopServer, CopyWar, ReDeploy, StartServer" description="WAR ファイルを WLS にデプロイする">
    <echo message="[${projectName}] Deploy Finished."/>
  </target>
  
  <target name="Refresh" description="プロジェクトをリフレッシュする">
    <eclipse.refreshLocal resource="${projectName}" depth="infinite"/>
  </target>
  
  <target name="FullBuild" description="プロジェクトのフルビルド (コンパイル)">
    <eclipse.incrementalBuild project="${projectName}" kind="full"/>
    <!--
      - project 属性
        - 未指定の場合はワークスペースになる
      - kind 属性
        - incremental (デフォルト)
        - full
        - clean
      フルビルドすることで classes ディレクトリに class ファイルがコンパイルされる
      src ディレクトリにある XML ファイルなども classes ディレクトリに格納される
    -->
  </target>
  
  <target name="CreateWar" description="WAR ファイルを作成する">
    <!-- 生成済の WAR ファイルを削除しておく -->
    <delete file="${warPath}" verbose="true"/>
    <!-- WAR ファイルを生成する -->
    <war destfile="${warPath}" webxml="${base}/WEB-INF/web.xml">
      <!-- クラスパスに追加するライブラリのルートディレクトリ -->
      <lib dir="${base}/WEB-INF/lib"/>
      <!-- クラスファイルがあるディレクトリ -->
      <classes dir="${base}/WEB-INF/classes"/>
      <!-- 除外するファイル・ディレクトリの指定 -->
      <fileset dir="${base}">
        <exclude name="*"/>                 <!-- ルート直下のファイルは不要なので除外する -->
        <exclude name="**/.settings/**/*"/> <!-- Eclipse の設定ファイルがあるディレクトリ配下 -->
        <exclude name="**/.settings"/>      <!-- ディレクトリ自体を除外するために記述 -->
        <exclude name="**/src/**/*"/>       <!-- src ディレクトリを除外 (Eclipse 上で生成するとディレクトリツリーが含まれるため) -->
        <exclude name="**/src"/>
        <exclude name="**/work/**/*"/>      <!-- Tomcat が JSP をコンパイルしたものが置かれるディレクトリ -->
        <exclude name="**/work"/>
        <!--
          - .svn ディレクトリなどはデフォルト除外パターンで除外される。
            除外したくない場合は war 要素に defaultexcludes="no" を指定する。
          - /META-INF/MANIFEST.MF がプロジェクトディレクトリ内になくても Ant ビルド時に自動生成される
        -->
      </fileset>
    </war>
  </target>
  
  <target name="StopServer" description="アプリケーションサーバを停止する">
    <wls action="stop" verbose="true" debug="true"
         adminurl="${adminUrl}" user="${userName}" password="${password}" targets="${target}" name="${aplName}"/>
  </target>
  
  <target name="CopyWar" description="WAR ファイルをデプロイ用ディレクトリへコピーする">
    <copy file="${warPath}" tofile="${uploadPath}" preservelastmodified="true" overwrite="true" verbose="true"/>
  </target>
  
  <target name="ReDeploy" description="WAR ファイルをデプロイする">
    <wls action="redeploy" verbose="true" debug="true"
         adminurl="${adminUrl}" user="${userName}" password="${password}" targets="${target}" name="${aplName}" source="${uploadPath}"/>
  </target>
  
  <target name="StartServer" description="アプリケーションサーバを起動する">
    <wls action="start" verbose="true" debug="true"
         adminurl="${adminUrl}" user="${userName}" password="${password}" targets="${target}" name="${aplName}"/>
  </target>
  
</project>
