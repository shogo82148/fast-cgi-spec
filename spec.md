# FastCGI Specification

Mark R. Brown  
Open Market, Inc.

Document Version: 1.0  
29 April 1996

**Copyright © 1996 Open Market, Inc. 245 First Street, Cambridge, MA 02142 U.S.A.**  
**Tel: 617-621-9500 Fax: 617-621-1703 URL: http://www.openmarket.com/**

**$Id: fcgi-spec.html,v 1.4 2002/02/25 00:42:59 robs Exp $**

* * *

  1. [Introduction](#1-introduction)
  2. [Initial process state](#2-initial-process-state)
      1. [Argument list](#21-argument-list)
      2. [File descriptors](#22-file-descriptors)
      3. [Environment variables](#23-environment-variables)
      4. [Other state](#24-other-state)
  3. [Protocol basics](#3-protocol-basics)
      1. [Notation](#31-notation)
      2. [Accepting transport connections](#32-accepting-transport-connections)
      3. [Records](#33-records)
      4. [Name-Value pairs](#34-name-value-pairs)
      5. [Closing transport connections](#35-closing-transport-connections)
  4. [Management record types](#4-management-record-types)
      1. [`FCGI_GET_VALUES`, `FCGI_GET_VALUES_RESULT`](#41-fcgi_get_values-fcgi_get_values_result)
      2. [`FCGI_UNKNOWN_TYPE`](#42-fcgi_unknown_type)
  5. [Application record types](#5-application-record-types)
      1. [`FCGI_BEGIN_REQUEST`](#51-fcgi_begin_request)
      2. [Name-Value pair streams: `FCGI_PARAMS`](#52-name-value-pair-streams-fcgi_params)
      3. [Byte streams: `FCGI_STDIN`, `FCGI_DATA`, `FCGI_STDOUT`, `FCGI_STDERR`](#53-byte-streams-fcgi_stdin-fcgi_data-fcgi_stdout-fcgi_stderr)
      4. [`FCGI_ABORT_REQUEST`](#54-fcgi_abort_request)
      5. [`FCGI_END_REQUEST`](#55-fcgi_end_request)
  6. [Roles](#6-roles)
      1. [Role protocols](#61-role-protocols)
      2. [Responder](#62-responder)
      3. [Authorizer](#63-authorizer)
      4. [Filter](#64-filter)
  7. [Errors](#7-errors)
  8. [Types and constants](#8-types-and-constants)
  9. [References](#9-references)
  10. [Table: properties of the record types](#a-table-properties-of-the-record-types)
  11. [Typical protocol message flow](#b-typical-protocol-message-flow)

* * *

## 1. Introduction

FastCGIはCGIのオープンな拡張機能で、WebサーバーAPIのペナルティなしにすべてのインターネットアプリケーションに高いパフォーマンスを提供します。
<!-- FastCGI is an open extension to CGI that provides high performance for all Internet applications without the penalties of Web server APIs. -->

この仕様の目的は、アプリケーションの観点から、FastCGI アプリケーションと FastCGI をサポートする Web サーバー間のインターフェイスを定めることです。 アプリケーション管理機能など、FastCGI に関連する多くの Web サーバー機能は、アプリケーションと Web サーバーのインターフェイスとは無関係であり、ここでは説明しません。
<!-- This specification has narrow goal: to specify, from an application perspective, the interface between a FastCGI application and a Web server that supports FastCGI. Many Web server features related to FastCGI, e.g. application management facilities, have nothing to do with the application to Web server interface, and are not described here. -->

この仕様は、Unix (より正確には、Berkeley Sockets をサポートする POSIX システム用) のためのものです。この仕様の大部分は、バイト順に依存しない単純な通信プロトコルであり、他のシステムにも拡張される予定です。
<!-- This specification is for Unix (more precisely, for POSIX systems that support Berkeley Sockets). The bulk of the specification is a simple communications protocol that is independent of byte ordering and will extend to other systems. -->

FastCGIを従来のUnixのCGI/1.1の実装と比較しながら紹介します。FastCGI は長寿命のアプリケーションプロセス、つまり *アプリケーションサーバ* をサポートするように設計されています。これは、アプリケーションプロセスを起動し、それを使用してリクエストに応答し、それを終了させる従来の Unix の CGI/1.1 の実装と比べて大きな違いです。
<!-- We'll introduce FastCGI by comparing it with conventional Unix implementations of CGI/1.1. FastCGI is designed to support long-lived application processes, i.e. *application servers*. That's a major difference compared with conventional Unix implementations of CGI/1.1, which construct an application process, use it respond to one request, and have it exit. -->

FastCGI プロセスの初期状態は CGI/1.1 プロセスの初期状態よりも質素です。FastCGI プロセスは何かに接続されて生活を始めないため、従来のオープンファイル `stdin`, `stdout`, `stderr` を持っておらず、環境変数を通して多くの情報を受け取りません。FastCGI プロセスの初期状態の重要な部分は、Web サーバーからの接続を受け入れるリスニングソケットです。
<!-- The initial state of a FastCGI process is more spartan than the initial state of a CGI/1.1 process, because the FastCGI process doesn't begin life connected to anything. It doesn't have the conventional open files `stdin`, `stdout`, and `stderr`, and it doesn't receive much information through environment variables. The key piece of initial state in a FastCGI process is a listening socket, through which it accepts connections from a Web server. -->

FastCGI プロセスがリスニングソケットで接続を受け入れた後、プロセスはデータを送受信するための簡単なプロトコルを実行します。このプロトコルには2つの目的があります。まず、プロトコルは、複数の独立した FastCGI 要求間の単一のトランスポート接続を多重化します。これは、イベント駆動型またはマルチスレッドプログラミング技術を使用して同時要求を処理できるアプリケーションをサポートします。第二に、各リクエスト内で、プロトコルは各方向に複数の独立したデータストリームを提供します。この方法では、CGI/1.1 のように別々のパイプを必要とするのではなく、例えば `stdout` と `stderr` の両方のデータがアプリケーションから Web サーバーへの単一のトランスポート接続を通過します。
<!-- After a FastCGI process accepts a connection on its listening socket, the process executes a simple protocol to receive and send data. The protocol serves two purposes. First, the protocol multiplexes a single transport connection between several independent FastCGI requests. This supports applications that are able to process concurrent requests using event-driven or multi-threaded programming techniques. Second, within each request the protocol provides several independent data streams in each direction. This way, for instance, both `stdout` and `stderr` data pass over a single transport connection from the application to the Web server, rather than requiring separate pipes as with CGI/1.1. -->

FastCGI アプリケーションは、いくつかの明確に定義された *Role* のうちの 1 つを果たします。最もよく知られているのは *Responder* の役割で、アプリケーションは HTTP リクエストに関連するすべての情報を受け取り、HTTP レスポンスを生成します。2 番目の役割は *Authorizer* で、アプリケーションは HTTP リクエストに関連するすべての情報を受け取り、承認/非承認の決定を生成します。三つ目の役割は *Filter* で、アプリケーションは HTTP リクエストに関連付けられたすべての情報に加えて、 Web サーバに保存されたファイルからの余分なデータストリームを受け取り、HTTP 応答としてデータストリームの "フィルタリングされた" バージョンを生成します。このフレームワークは拡張可能なので、後でより多くの FastCGI を定義することができます。
<!-- A FastCGI application plays one of several well-defined *roles*. The most familiar is the *Responder* role, in which the application receives all the information associated with an HTTP request and generates an HTTP response; that's the role CGI/1.1 programs play. A second role is *Authorizer*, in which the application receives all the information associated with an HTTP request and generates an authorized/unauthorized decision. A third role is *Filter*, in which the application receives all the information associated with an HTTP request, plus an extra stream of data from a file stored on the Web server, and generates a "filtered" version of the data stream as an HTTP response. The framework is extensible so that more FastCGI can be defined later. -->

この仕様の残りの部分では、「FastCGI アプリケーション」、「アプリケーション プロセス」、または「アプリケーション サーバー」という用語は、混乱を招かないように「アプリケーション」と略されています。
<!-- In the remainder of this specification the terms "FastCGI application," "application process," or "application server" are abbreviated to "application" whenever that won't cause confusion. -->

## 2. Initial process state

### 2.1. Argument list

デフォルトでは、ウェブサーバは、実行ファイルのパス名の最後のコンポーネントとなるアプリケーションの名前という単一の要素を含む引数リストを作成します。ウェブサーバは、異なるアプリケーション名や、より精巧な引数リストを指定する方法を提供することができます。
<!-- By default the Web server creates an argument list containing a single element, the name of the application, taken to be the last component of the executable's path name. The Web server may provide a way to specify a different application name, or a more elaborate argument list. -->

ウェブサーバによって実行されるファイルはインタプリタファイル( `#!` で始まるテキストファイル)かもしれないことに注意してください。
<!-- Note that the file executed by the Web server might be an interpreter file (a text file that starts with the characters `#!`), in which case the application's argument list is constructed as described in the `execve` manpage. -->

### 2.2. File descriptors

ウェブサーバはアプリケーションの実行開始時に `FCGI_LISTENSOCK_FILENO` という単一のファイルディスクリプタを開きます。このディスクリプタはWebサーバが作成したリスニングソケットを参照します。
<!-- The Web server leaves a single file descriptor, `FCGI_LISTENSOCK_FILENO`, open when the application begins execution. This descriptor refers to a listening socket created by the Web server. -->

アプリケーションの実行開始時に `STDIN_FILENO` は `FCGI_LISTENSOCK_FILENO` に設定され、標準ディスクリプタ `STDOUT_FILENO` と `STDERR_FILENO` は閉じられます。アプリケーションがCGIかFastCGIかを判断するための信頼できる方法は、`getpeername(FCGI_LISTENSOCK_FILENO)` を呼び出すことです。
<!-- `FCGI_LISTENSOCK_FILENO` equals `STDIN_FILENO`. The standard descriptors `STDOUT_FILENO` and `STDERR_FILENO` are closed when the application begins execution. A reliable method for an application to determine whether it was invoked using CGI or FastCGI is to call `getpeername(FCGI_LISTENSOCK_FILENO)`, which returns -1 with `errno` set to `ENOTCONN` for a FastCGI application. -->

Webサーバが信頼性の高いトランスポート、Unixストリームパイプ(`AF_UNIX`)やTCP/IP(`AF_INET`)を選択するかどうかは、`FCGI_LISTENSOCK_FILENO` ソケットの内部状態に暗黙的に反映されます。
<!-- The Web server's choice of reliable transport, Unix stream pipes (`AF_UNIX`) or TCP/IP (`AF_INET`), is implicit in the internal state of the `FCGI_LISTENSOCK_FILENO` socket. -->

### 2.3. Environment variables

ウェブサーバは、環境変数を使用してアプリケーションにパラメータを渡すことができます。本仕様では、そのような環境変数の一つである `FCGI_WEB_SERVER_ADDRS` を定義しています。Webサーバは、`PATH`変数のような他の環境変数をバインドする方法を提供することができます。
<!-- The Web server may use environment variables to pass parameters to the application. This specification defines one such variable, `FCGI_WEB_SERVER_ADDRS`; we expect more to be defined as the specification evolves. The Web server may provide a way to bind other environment variables, such as the `PATH` variable. -->

### 2.4. Other state

ウェブサーバは、アプリケーションの初期プロセス状態の他の構成要素、例えば、プロセスの優先度、ユーザID、グループID、ルートディレクトリ、作業ディレクトリを指定する方法を提供してもかまいません。
<!-- The Web server may provide a way to specify other components of an application's initial process state, such as the priority, user ID, group ID, root directory, and working directory of the process. -->

## 3. Protocol basics

### 3.1. Notation

プロトコルメッセージのフォーマットを定義するためにC言語の表記法を使用しています。すべての構造体要素は `unsigned char` 型で定義されており、ISO C コンパイラがパディングなしで明らかな方法でレイアウトするように配置されています。構造体で定義された最初のバイトが最初に送信され、2番目のバイトが2番目に送信されます。
<!-- We use C language notation to define protocol message formats. All structure elements are defined in terms of the `unsigned char` type, and are arranged so that an ISO C compiler lays them out in the obvious manner, with no padding. The first byte defined in the structure is transmitted first, the second byte second, etc. -->

定義を省略するために2つの規則を使用します。
<!-- We use two conventions to abbreviate our definitions. -->

まず、隣接する２つの構造体の構成要素が、接尾辞「`B1`」と「`B0`」を除いて同一の名前を持つ場合、それは、2つの構成要素を、 `B1<<8 + B0` のように計算された単一の数として見ることができることを意味します。この単一の数の名前は構成要素の名前から接尾辞を差し引いたものです。この規則は、2バイト以上で表現された数値を扱うための明白な方法を一般化します。
<!-- First, when two adjacent structure components are named identically except for the suffixes "`B1`" and "`B0`", it means that the two components may be viewed as a single number, computed as `B1<<8 + B0`. The name of this single number is the name of the components, minus the suffixes. This convention generalizes in an obvious way to handle numbers represented in more than two bytes. -->

つぎに C の `struct` を拡張して、次のような形式を可能にします。
<!-- Second, we extend C `structs` to allow the form -->

```c
struct {
    unsigned char mumbleLengthB1;
    unsigned char mumbleLengthB0;
    ... /* other stuff */
    unsigned char mumbleData[mumbleLength];
};
```

長さが可変な構造を意味し、ここで、構成要素の長さは、指示された以前の構成要素または構成要素の値によって決定されます。
<!-- meaning a structure of varying length, where the length of a component is determined by the values of the indicated earlier component or components. -->

### 3.2. Accepting transport connections

FastCGIアプリケーションはファイルディスクリプタ `FCGI_LISTENSOCK_FILENO` が参照しているソケットに対して `accept()` を呼び出し、新しいトランスポート接続を受け付けます。 `accept()` が成功し、環境変数 `FCGI_WEB_SERVER_ADDRS` がバインドされている場合、アプリケーションは以下の特殊な処理を即座に行います。
<!-- A FastCGI application calls `accept()` on the socket referred to by file descriptor `FCGI_LISTENSOCK_FILENO` to accept a new transport connection. If the `accept()` succeeds, and the `FCGI_WEB_SERVER_ADDRS` environment variable is bound, the application application immediately performs the following special processing: -->

  * `FCGI_WEB_SERVER_ADDRS`. 値はウェブサーバで有効なIPアドレスのリストである。
<!--  * `FCGI_WEB_SERVER_ADDRS`: The value is a list of valid IP addresses for the Web server. -->

`FCGI_WEB_SERVER_ADDRS` がバインドされていた場合、アプリケーションは接続先のIPアドレスがリストに登録されているかどうかを調べます。チェックに失敗した場合(TCP/IPトランスポートを使っていない可能性も含めて)、アプリケーションは接続を閉じることで応答します。
<!-- If `FCGI_WEB_SERVER_ADDRS` was bound, the application checks the peer IP address of the new connection for membership in the list. If the check fails (including the possibility that the connection didn't use TCP/IP transport), the application responds by closing the connection. -->

`FCGI_WEB_SERVER_ADDRS` はカンマ区切りのIPアドレスのリストです。各IPアドレスは、`[0..255]` の範囲内の数字4個を小数点記号で区切ったもので表します。したがって、この環境変数の合法なバインディングは `FCGI_WEB_SERVER_ADDRS=199.170.183.28,199.170.183.71` のようなものになります。
<!-- `FCGI_WEB_SERVER_ADDRS` is expressed as a comma-separated list of IP addresses. Each IP address is written as four decimal numbers in the range `[0..255]` separated by decimal points. So one legal binding for this variable is `FCGI_WEB_SERVER_ADDRS=199.170.183.28,199.170.183.71`. -->

アプリケーションは複数の同時トランスポート接続を受け入れることができますが、そうする必要はありません。
<!-- An application may accept several concurrent transport connections, but it need not do so. -->

### 3.3. Records

アプリケーションは単純なプロトコルを使って Web サーバからのリクエストを実行します。プロトコルの詳細はアプリケーションの Role に依存しますが、大まかに言えば、Web サーバーが最初にパラメータやその他のデータをアプリケーションに送り、次にアプリケーションが結果データを Web サーバーに送り、最後にアプリケーションが Web サーバーにリクエストが完了を示すデータを送ります。
<!-- Applications execute requests from a Web server using a simple protocol. Details of the protocol depend upon the application's role, but roughly speaking the Web server first sends parameters and other data to the application, then the application sends result data to the Web server, and finally the application sends the Web server an indication that the request is complete. -->

トランスポート接続を流れるすべてのデータは、 *FastCGIレコード* で運ばれます。FastCGI レコードは 2 つのことを達成します。まず、レコードは、複数の独立した FastCGI 要求間でトランスポート接続を多重化します。この多重化は、イベント駆動型またはマルチスレッドのプログラミング技術を使用して同時要求を処理できるアプリケーションをサポートします。第二に、レコードは、1 つのリクエスト内で各方向に複数の独立したデータストリームを提供します。この方法では、例えば、`stdout` と `stderr` の両方のデータを、別々の接続を必要とするのではなく、アプリケーションから Web サーバーへの単一のトランスポート接続を介して渡すことができます。
<!-- All data that flows over the transport connection is carried in *FastCGI records*. FastCGI records accomplish two things. First, records multiplex the transport connection between several independent FastCGI requests. This multiplexing supports applications that are able to process concurrent requests using event-driven or multi-threaded programming techniques. Second, records provide several independent data streams in each direction within a single request. This way, for instance, both `stdout` and `stderr` data can pass over a single transport connection from the application to the Web server, rather than requiring separate connections. -->

```c
typedef struct {
    unsigned char version;
    unsigned char type;
    unsigned char requestIdB1;
    unsigned char requestIdB0;
    unsigned char contentLengthB1;
    unsigned char contentLengthB0;
    unsigned char paddingLength;
    unsigned char reserved;
    unsigned char contentData[contentLength];
    unsigned char paddingData[paddingLength];
} FCGI_Record;
```

FastCGI レコードは、固定長の接頭辞と可変数のコンテンツとパディングバイトで構成されています。レコードには 7 つのコンポーネントが含まれます。
<!-- A FastCGI record consists of a fixed-length prefix followed by a variable number of content and padding bytes. A record contains seven components: -->

  * `version`: FastCGI プロトコルのバージョンを表します。この仕様は `FCGI_VERSION_1` について書かれています。
<!--  * `version`: Identifies the FastCGI protocol version. This specification documents `FCGI_VERSION_1`. -->

  * `type`: FastCGI レコードタイプ、つまりレコードが実行する一般的な機能を表します。特定のレコードタイプとその機能については、後のセクションで詳しく説明します。

<!--  * `type`: Identifies the FastCGI record type, i.e. the general function that the record performs. Specific record types and their functions are detailed in later sections. -->

  * `requestId`: レコードが属する *FastCGI リクエスト* を表します。
<!--  * `requestId`: Identifies the *FastCGI request* to which the record belongs. -->

  * `contentLength`: レコードの `contentData` コンポーネントに含まれるバイト数。
<!--  * `contentLength`: The number of bytes in the `contentData` component of the record.-->

  * `paddingLength`: レコードの `paddingData` コンポーネントに含まれるバイト数。
<!--  * `paddingLength`: The number of bytes in the `paddingData` component of the record. -->

  * `contentData`: 0から65535バイトのデータ、レコードタイプに応じて解釈されます。
<!--  * `contentData`: Between 0 and 65535 bytes of data, interpreted according to the record type.-->

  * `paddingData`: 0～255バイトのデータ、無視されます。
<!--  * `paddingData`: Between 0 and 255 bytes of data, which are ignored.-->

定数のFastCGIレコードを指定するために、くだけたCの `struct` 初期化構文を使用します。 `version` コンポーネントを省略し、パディングを無視し、`requestId` を数値として扱います。したがって、`{FCGI_END_REQUEST, 1, {FCGI_REQUEST_COMPLETE,0}}`は、`type == FCGI_END_REQUEST`、`requestId == 1`、`contentData == {FCGI_REQUEST_COMPLETE,0}`のレコードです。
<!-- We use a relaxed C `struct` initializer syntax to specify constant FastCGI records. We omit the `version` component, ignore padding, and treat `requestId` as a number. Thus `{FCGI_END_REQUEST, 1, {FCGI_REQUEST_COMPLETE,0}}` is a record with `type == FCGI_END_REQUEST`, `requestId == 1`, and `contentData == {FCGI_REQUEST_COMPLETE,0}`. -->

#### Padding

このプロトコルでは、送信者が送信するレコードをパディングすることができ、受信者は `paddingLength` を解釈して `paddingData` をスキップしなくてはなりません。パディングすることで、送信者はより効率的な処理のためにデータを整列させておくことができます。Xウィンドウシステムのプロトコルでの経験から、このようなアラインメントの性能上の利点が示されています。
<!-- The protocol allows senders to pad the records they send, and requires receivers to interpret the `paddingLength` and skip the `paddingData`. Padding allows senders to keep data aligned for more efficient processing. Experience with the X window system protocols shows the performance benefit of such alignment. -->

レコードは8バイトの倍数の境界線上に配置することを推奨します。`FCGI_Record` の固定長部分は8バイトです。
<!-- We recommend that records be placed on boundaries that are multiples of eight bytes. The fixed-length portion of a `FCGI_Record` is eight bytes. -->

#### Managing request IDs

Web サーバーは FastCGI リクエスト ID を再利用します。アプリケーションは、指定されたトランスポート接続上の各リクエスト ID の現在の状態を追跡します。リクエストID `R` は、アプリケーションがレコード `{FCGI_BEGIN_REQUEST, R, ...}` を受信するとアクティブになり、アプリケーションがレコード `{FCGI_END_REQUEST, R, ...}` を Web サーバーに送信すると非アクティブになります。
<!-- The Web server re-uses FastCGI request IDs; the application keeps track of the current state of each request ID on a given transport connection. A request ID `R` becomes active when the application receives a record `{FCGI_BEGIN_REQUEST, R, ...}` and becomes inactive when the application sends a record `{FCGI_END_REQUEST, R, ...}` to the Web server.-->

リクエストID `R` が非アクティブな場合、アプリケーションは `requestId == R` のレコードを無視します。ただし、先ほど説明したとおり `FCGI_BEGIN_REQUEST` レコードは除きます。
<!-- While a request ID `R` is inactive, the application ignores records with `requestId == R`, except for `FCGI_BEGIN_REQUEST` records as just described. -->

Web サーバーは FastCGI リクエスト ID を小さくしようとします。そうすることで、アプリケーションは長い配列やハッシュテーブルではなく、短い配列を使ってリクエスト ID の状態を追跡することができます。アプリケーションには、一度に 1 つのリクエストのみを受け入れるオプションもあります。この場合、アプリケーションは単に現在のリクエストIDに対して受信する `requestId` の値をチェックします。
<!-- The Web server attempts to keep FastCGI request IDs small. That way the application can keep track of request ID states using a short array rather than a long array or a hash table. An application also has the option of accepting only one request at a time. In this case the application simply checks incoming `requestId` values against the current request ID. -->

#### Types of record types

FastCGI レコードタイプの分類には、2つの便利な方法があります。
<!-- There are two useful ways of classifying FastCGI record types. -->

最初の区別は、 *management* レコードと *application* レコードの間にあります。管理レコードには、アプリケーションのプロトコル機能に関する情報など、Webサーバーのリクエストに固有ではない情報が含まれています。アプリケーションレコードは特定のリクエストに関する情報を含み、`requestId` コンポーネントによって識別されます。
<!-- The first distinction is between *management* records and *application* records. A management record contains information that is not specific to any Web server request, such as information about the protocol capabilities of the application. An application record contains information about a particular request, identified by the `requestId` component. -->

管理レコードは `requestId` の値が0で、*nullリクエストID*とも呼ばれる。アプリケーションレコードは `requestId` の値が0以外のものを持つ。
<!-- Management records have a `requestId` value of zero, also called the *null request ID*. Application records have a nonzero `requestId`.-->

2つ目の区別は、*離散*レコードと*ストリーム*レコードの間にあります。離散レコードは、それ自体が意味のあるデータ単位を含んでいます。ストリームレコードは*ストリーム*の一部であり、ストリーム型の0個以上の空でないレコード(`length != 0`)の連続したものであり、その後にストリーム型の空のレコード(`length == 0`)が続きます。ストリームのレコードの `contentData` 構成要素は、連結されるとバイト列を形成し、このバイト列がストリームの値となる。このバイト列がストリームの値となります。したがって、ストリームの値は、レコードの数や空ではないレコード間でのバイトの分割方法に依存しません。
<!-- The second distinction is between *discrete* and *stream* records. A discrete record contains a meaningful unit of data all by itself. A stream record is part of a *stream*, i.e. a series of zero or more non-empty records (`length != 0`) of the stream type, followed by an empty record (`length == 0`) of the stream type. The `contentData` components of a stream's records, when concatenated, form a byte sequence; this byte sequence is the value of the stream. Therefore the value of a stream is independent of how many records it contains or how its bytes are divided among the non-empty records. -->

これらの 2 つの分類は独立しています。このバージョンの FastCGI プロトコルで定義されているレコードタイプのうち、すべての管理レコードタイプは離散レコードタイプであり、ほぼすべてのアプリケーションレコードタイプはストリームレコードタイプです。しかし、3つのアプリケーションレコードタイプは離散レコードタイプであり、ストリームである管理レコードタイプをプロトコルのいくつかの後のバージョンで定義することを妨げるものは何もありません。
<!-- These two classifications are independent. Among the record types defined in this version of the FastCGI protocol, all management record types are also discrete record types, and nearly all application record types are stream record types. But three application record types are discrete, and nothing prevents defining a management record type that's a stream in some later version of the protocol. -->

### 3.4. Name-Value pairs

多くの役割において、FastCGI アプリケーションは、さまざまな数の可変長値を読み書きする必要があります。そのため、名前と値のペアをエンコードするための標準フォーマットを採用すると便利です。
<!-- In many of their roles, FastCGI applications need to read and write varying numbers of variable-length values. So it is useful to adopt a standard format for encoding a name-value pair. -->

FastCGI は、名前と値のペアを、名前の長さ、値の長さ、名前の長さ、値の長さの順に送信します。127バイト以下の長さは1バイトでエンコードできますが、それ以上の長さは常に4バイトでエンコードされます。
<!-- FastCGI transmits a name-value pair as the length of the name, followed by the length of the value, followed by the name, followed by the value. Lengths of 127 bytes and less can be encoded in one byte, while longer lengths are always encoded in four bytes: -->

```c
typedef struct {
    unsigned char nameLengthB0;  /* nameLengthB0  >> 7 == 0 */
    unsigned char valueLengthB0; /* valueLengthB0 >> 7 == 0 */
    unsigned char nameData[nameLength];
    unsigned char valueData[valueLength];
} FCGI_NameValuePair11;

typedef struct {
    unsigned char nameLengthB0;  /* nameLengthB0  >> 7 == 0 */
    unsigned char valueLengthB3; /* valueLengthB3 >> 7 == 1 */
    unsigned char valueLengthB2;
    unsigned char valueLengthB1;
    unsigned char valueLengthB0;
    unsigned char nameData[nameLength];
    unsigned char valueData[valueLength
            ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
} FCGI_NameValuePair14;

typedef struct {
    unsigned char nameLengthB3;  /* nameLengthB3  >> 7 == 1 */
    unsigned char nameLengthB2;
    unsigned char nameLengthB1;
    unsigned char nameLengthB0;
    unsigned char valueLengthB0; /* valueLengthB0 >> 7 == 0 */
    unsigned char nameData[nameLength
            ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
    unsigned char valueData[valueLength];
} FCGI_NameValuePair41;

typedef struct {
    unsigned char nameLengthB3;  /* nameLengthB3  >> 7 == 1 */
    unsigned char nameLengthB2;
    unsigned char nameLengthB1;
    unsigned char nameLengthB0;
    unsigned char valueLengthB3; /* valueLengthB3 >> 7 == 1 */
    unsigned char valueLengthB2;
    unsigned char valueLengthB1;
    unsigned char valueLengthB0;
    unsigned char nameData[nameLength
            ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
    unsigned char valueData[valueLength
            ((B3 & 0x7f) << 24) + (B2 << 16) + (B1 << 8) + B0];
} FCGI_NameValuePair44;
```

長さの最初のバイトの高次ビットは、長さのエンコーディングを示します。高次の0は1バイトエンコーディングを意味し、1は4バイトエンコーディングを意味します。
<!-- The high-order bit of the first byte of a length indicates the length's encoding. A high-order zero implies a one-byte encoding, a one a four-byte encoding. -->

この名前と値のペア形式により、送信者は追加の符号化を行わずにバイナリ値を送信することができ、受信者は大きな値でもすぐに正しいストレージ量を割り当てることができます。
<!-- This name-value pair format allows the sender to transmit binary values without additional encoding, and enables the receiver to allocate the correct amount of storage immediately even for large values. -->

### 3.5. Closing transport connections

Web サーバーはトランスポート接続の有効期間を制御します。ウェブサーバはリクエストがアクティブになっていないときに接続を閉じることができます。あるいは、ウェブサーバはアプリケーションにクローズ権限を委譲することができます (`FCGI_BEGIN_REQUEST` を参照してください)。この場合、アプリケーションは指定されたリクエストの終了時に接続を閉じます。
<!-- The Web server controls the lifetime of transport connections. The Web server can close a connection when no requests are active. Or the Web server can delegate close authority to the application (see `FCGI_BEGIN_REQUEST`). In this case the application closes the connection at the end of a specified request. -->

この柔軟性は、様々なアプリケーションスタイルに対応します。単純なアプリケーションは、一度に1つのリクエストを処理し、各リクエストに対して新しいトランスポート コネクションを受け入れる。より複雑なアプリケーションは、1つまたは複数のトランスポート接続上で、同時にリクエストを 処理し、トランスポート接続を長時間開いたままにします。
<!-- This flexibility accommodates a variety of application styles. Simple applications will process one request at a time and accept a new transport connection for each request. More complex applications will process concurrent requests, over one or multiple transport connections, and will keep transport connections open for long periods of time. -->

単純なアプリケーションは、応答の書き込みが終了したときにトランスポート接続を閉じることで、パフォーマンスが大幅に向上します。Web サーバーは、長寿命の接続のために接続寿命を制御する必要があります。
<!-- A simple application gets a significant performance boost by closing the transport connection when it has finished writing its response. The Web server needs to control the connection lifetime for long-lived connections. -->

アプリケーションが接続を閉じた場合、または接続が閉じたことが判明した場合、アプリケーションは新しい接続を開始します。
<!-- When an application closes a connection or finds that a connection has closed, the application initiates a new connection. -->

## 4. Management record types

### 4.1. `FCGI_GET_VALUES`, `FCGI_GET_VALUES_RESULT`

ウェブサーバは、アプリケーション内の特定の変数にクエリを実行することができます。サーバは通常、システム構成の特定の側面を自動化するために、アプリケーションの起動時にクエリを実行します。
<!-- The Web server can query specific variables within the application. The server will typically perform a query on application startup in order to to automate certain aspects of system configuration. -->

アプリケーションはクエリをレコード `{FCGI_GET_VALUES, 0, ...}` として受け取ります。レコード `FCGI_GET_VALUES` の `contentData` 部分には、空の値を持つ名前と値のペアのシーケンスが含まれています。
<!-- The application receives a query as a record `{FCGI_GET_VALUES, 0, ...}`. The `contentData` portion of a `FCGI_GET_VALUES` record contains a sequence of name-value pairs with empty values. -->

アプリケーションは `{FCGI_GET_VALUES_RESULT, 0, ...}` というレコードを送信して応答します。クエリに含まれている変数名がわからない場合は、その変数名を省略して応答します。
<!-- The application responds by sending a record `{FCGI_GET_VALUES_RESULT, 0, ...}` with the values supplied. If the application doesn't understand a variable name that was included in the query, it omits that name from the response. -->

`FCGI_GET_VALUES` は、自由に変数を設定できるように設計されています。初期値はサーバがアプリケーションや接続管理を行うための情報を提供します。
<!-- `FCGI_GET_VALUES` is designed to allow an open-ended set of variables. The initial set provides information to help the server perform application and connection management: -->

  * `FCGI_MAX_CONNS`: このアプリケーションが受け入れる同時接続の最大数を指定します。例えば、`"1"` や `"10"` のように。
<!--  * `FCGI_MAX_CONNS`: The maximum number of concurrent transport connections this application will accept, e.g. `"1"` or `"10"`. -->

  * `FCGI_MAX_REQS`: このアプリケーションが受け付ける同時リクエストの最大数、例えば `"1"` や `"50"` を指定します。
<!--  * `FCGI_MAX_REQS`: The maximum number of concurrent requests this application will accept, e.g. `"1"` or `"50"`. -->

  * `FCGI_MPXS_CONNS`: このアプリケーションが接続を多重化しない(つまり、各接続に対する同時リクエストを扱う)場合は `"0"`、そうでない場合は `"1"` となります。
<!--  * `FCGI_MPXS_CONNS`: `"0"` if this application does not multiplex connections (i.e. handle concurrent requests over each connection), `"1"` otherwise. -->

アプリケーションはいつでも `FCGI_GET_VALUES` レコードを受け取ることができます。アプリケーションのレスポンスは、アプリケーションには関係なく、FastCGI ライブラリにのみ関係していなければなりません。
<!-- An application may receive a `FCGI_GET_VALUES` record at any time. The application's response should not involve the application proper but only the FastCGI library. -->

### 4.2. `FCGI_UNKNOWN_TYPE`

管理レコードの型のセットは、将来のバージョンで大きくなる可能性があります。このため、このプロトコルには `FCGI_UNKNOWN_TYPE` という管理レコードが含まれています。アプリケーションが `T` 型が理解できない管理レコードを受け取った場合、アプリケーションは `{FCGI_UNKNOWN_TYPE, 0, {T}}` で応答する。
<!-- The set of management record types is likely to grow in future versions of this protocol. To provide for this evolution, the protocol includes the `FCGI_UNKNOWN_TYPE` management record. When an application receives a management record whose type `T` it does not understand, the application responds with `{FCGI_UNKNOWN_TYPE, 0, {T}}`. -->

レコード `FCGI_UNKNOWN_TYPE` の `contentData` コンポーネントの形式は以下の通りです。
<!-- The `contentData` component of a `FCGI_UNKNOWN_TYPE` record has the form: -->

```c
typedef struct {
    unsigned char type;    
    unsigned char reserved[7];
} FCGI_UnknownTypeBody;
```

タイプコンポーネントは、未認識管理レコードのタイプです。
<!-- The type component is the type of the unrecognized management record. -->

## 5. Application record types

### 5.1. `FCGI_BEGIN_REQUEST`

ウェブサーバは `FCGI_BEGIN_REQUEST` レコードを送信してリクエストを開始します。
<!-- The Web server sends a `FCGI_BEGIN_REQUEST` record to start a request. -->

レコード `FCGI_BEGIN_REQUEST` の `contentData` コンポーネントはフォームを持ちます。
<!-- The `contentData` component of a `FCGI_BEGIN_REQUEST` record has the form: -->

```c
typedef struct {
    unsigned char roleB1;
    unsigned char roleB0;
    unsigned char flags;
    unsigned char reserved[5];
} FCGI_BeginRequestBody;
```

`role`コンポーネントは、Webサーバがアプリケーションに期待する役割を設定します。現在定義されているロールは以下の通りです。
<!-- The `role` component sets the role the Web server expects the application to play. The currently-defined roles are: -->

  * `FCGI_RESPONDER`
  * `FCGI_AUTHORIZER`
  * `FCGI_FILTER`

役割については、以下の[Section 6](#roles)で詳しく説明します。
<!-- Roles are described in more detail in [Section 6](#roles) below. -->

`flags` コンポーネントには接続のシャットダウンを制御するビットが含まれる。
<!-- The `flags` component contains a bit that controls connection shutdown: -->

  * `flags & FCGI_KEEP_CONN`: If zero, the application closes the connection after responding to this request. If not zero, the application does not close the connection after responding to this request; the Web server retains responsibility for the connection.

### 5.2. Name-Value pair streams: `FCGI_PARAMS`

`FCGI_PARAMS` は、ウェブサーバからアプリケーションに名前と値のペアを送信する際に用いられるストリームレコード型である。名前と値のペアは、指定された順序ではなく、次から次へとストリームに送られる。
<!-- `FCGI_PARAMS` is a stream record type used in sending name-value pairs from the Web server to the application. The name-value pairs are sent down the stream one after the other, in no specified order. -->

### 5.3. Byte streams: `FCGI_STDIN`, `FCGI_DATA`, `FCGI_STDOUT`, `FCGI_STDERR`

`FCGI_STDIN` は、ウェブサーバからアプリケーションに任意のデータを送信する際に用いられるストリームレコード型です。`FCGI_DATA` は2番目のストリームレコード型で、追加のデータをアプリケーションに送信する際に利用します。
<!-- `FCGI_STDIN` is a stream record type used in sending arbitrary data from the Web server to the application. `FCGI_DATA` is a second stream record type used to send additional data to the application. -->

`FCGI_STDOUT` と `FCGI_STDERR` は、アプリケーションからウェブサーバに任意のデータとエラーデータをそれぞれ送信するためのストリームレコード型です。
<!-- `FCGI_STDOUT` and `FCGI_STDERR` are stream record types for sending arbitrary data and error data respectively from the application to the Web server. -->

### 5.4. `FCGI_ABORT_REQUEST`

ウェブサーバは `FCGI_ABORT_REQUEST` レコードを送信してリクエストを中止します。アプリケーションは `{FCGI_ABORT_REQUEST, R}` を受信した後、できるだけ早く `{FCGI_END_REQUEST, R, {FCGI_REQUEST_COMPLETE, appStatus}}` で応答します。これはアプリケーションからの応答であり、FastCGI ライブラリからの低レベルの確認ではありません。
<!-- The Web server sends a `FCGI_ABORT_REQUEST` record to abort a request. After receiving `{FCGI_ABORT_REQUEST, R}`, the application responds as soon as possible with `{FCGI_END_REQUEST, R, {FCGI_REQUEST_COMPLETE, appStatus}}`. This is truly a response from the application, not a low-level acknowledgement from the FastCGI library. -->

FastCGI 要求がそのクライアントの代わりに実行されている間に HTTP クライアントがトランスポート接続を閉じると、Web サーバーは FastCGI 要求をアボートします。ほとんどの FastCGI 要求は応答時間が短く、クライアントが遅い場合は Web サーバーが出力バッファリングを提供します。しかし、FastCGI アプリケーションは、他のシステムとの通信やサーバープッシュの実行が遅れることがあります。
<!-- A Web server aborts a FastCGI request when an HTTP client closes its transport connection while the FastCGI request is running on behalf of that client. The situation may seem unlikely; most FastCGI requests will have short response times, with the Web server providing output buffering if the client is slow. But the FastCGI application may be delayed communicating with another system, or performing a server push. -->

Web サーバーがトランスポート接続上でリクエストを多重化していない場合、Web サーバーはリクエストの トランスポート接続を閉じることでリクエストを中止することができます。しかし、リクエストが多重化されている場合、トランスポート接続を閉じることは、その接続上のすべてのリクエス トを中止するという不幸な効果をもたらします。
<!-- When a Web server is not multiplexing requests over a transport connection, the Web server can abort a request by closing the request's transport connection. But with multiplexed requests, closing the transport connection has the unfortunate effect of aborting all the requests on the connection. -->

### 5.5. `FCGI_END_REQUEST`

アプリケーションは `FCGI_END_REQUEST` レコードを送信してリクエストを終了させます。
<!-- The application sends a `FCGI_END_REQUEST` record to terminate a request, either because the application has processed the request or because the application has rejected the request. -->

レコード `FCGI_END_REQUEST` の `contentData` コンポーネントの形式は以下の通りです。
<!-- The `contentData` component of a `FCGI_END_REQUEST` record has the form: -->

```c
typedef struct {
    unsigned char appStatusB3;
    unsigned char appStatusB2;
    unsigned char appStatusB1;
    unsigned char appStatusB0;
    unsigned char protocolStatus;
    unsigned char reserved[3];
} FCGI_EndRequestBody;
```

コンポーネント `appStatus` はアプリケーションレベルのステータスコードである。各ロールは `appStatus` の使い方を文書化する。
<!-- The `appStatus` component is an application-level status code. Each role documents its usage of `appStatus`. -->

`protocolStatus` コンポーネントはプロトコルレベルのステータスコードである。
<!-- The `protocolStatus` component is a protocol-level status code; the possible `protocolStatus` values are: -->

  * `FCGI_REQUEST_COMPLETE`: リクエストの正常終了。
<!--  * `FCGI_REQUEST_COMPLETE`: normal end of request. -->

  * `FCGI_CANT_MPX_CONN`: 新しいリクエストを拒否しました。これは、Webサーバが1つの接続に対して同時に1つのリクエストを処理するように設計されたアプリケーションに1つの接続を介して同時にリクエストを送信した場合に発生します。
<!--  * `FCGI_CANT_MPX_CONN`: rejecting a new request. This happens when a Web server sends concurrent requests over one connection to an application that is designed to process one request at a time per connection. -->

  * `FCGI_OVERLOADED`: 新しいリクエストを拒否しています。これは、アプリケーションがデータベース接続などのリソースを使い果たしたときに発生します。
<!--  * `FCGI_OVERLOADED`: rejecting a new request. This happens when the application runs out of some resource, e.g. database connections. -->

  * `FCGI_UNKNOWN_ROLE`: 新規リクエストを拒否しました。これは、Webサーバがアプリケーションにとって未知のロールを指定した場合に発生します。
<!--  * `FCGI_UNKNOWN_ROLE`: rejecting a new request. This happens when the Web server has specified a role that is unknown to the application. -->

## 6. Roles

### 6.1. Role protocols

ロール・プロトコルは、アプリケーション・レコード・タイプを持つレコードのみを含みます。これらのプロトコルは、ストリームを使用して基本的にすべてのデータを転送します。
<!-- Role protocols only include records with application record types. They transfer essentially all data using streams. -->

プロトコルの信頼性を高め、アプリケーションのプログラミングを簡単にするために、ロールプロトコルは*ほぼシーケンシャルマーシャリング*を使用するように設計されています。厳密にシーケンシャルマーシャリングを使用したプロトコルでは、アプリケーションは、最初の入力を受信し、次に2番目の入力を受信するなど、すべてを受信するまで受信します。同様に、アプリケーションは最初の出力を送信し、次に2番目の出力を送信します。入力は互いにインターリーブされず、出力は互いにインターリーブされない。
<!-- To make the protocols reliable and to simplify application programming, role protocols are designed to use *nearly sequential marshalling*. In a protocol with strictly sequential marshalling, the application receives its first input, then its second, etc. until it has received them all. Similarly, the application sends its first output, then its second, etc. until it has sent them all. Inputs are not interleaved with each other, and outputs are not interleaved with each other. -->

シーケンシャルマーシャリングルールは FastCGI ロールによっては制限が強すぎます。そのため、`FCGI_STDOUT` と `FCGI_STDERR` の両方を使用するロールプロトコルでは、これらの二つのストリームをインターリーブすることができます。
<!-- The sequential marshalling rule is too restrictive for some FastCGI roles, because CGI programs can write to both stdout and stderr without timing restrictions. So role protocols that use both `FCGI_STDOUT` and `FCGI_STDERR` allow these two streams to be interleaved. -->

すべてのロールプロトコルは `FCGI_STDERR` ストリームを使用しますが、従来のアプリケーションプログラミングで `stderr` が使用されている方法と同じように `FCGI_STDERR` ストリームを使用します。 `FCGI_STDERR` ストリームの使用は常にオプションです。アプリケーションがエラーを報告する必要がない場合は、`FCGI_STDERR` レコードを何も送信しないか、長さ 0 の `FCGI_STDERR` レコードを 1 つ送信します。
<!-- All role protocols use the `FCGI_STDERR` stream just the way `stderr` is used in conventional applications programming: to report application-level errors in an intelligible way. Use of the `FCGI_STDERR` stream is always optional. If an application has no errors to report, it sends either no `FCGI_STDERR` records or one zero-length `FCGI_STDERR` record. -->

ロールプロトコルが `FCGI_STDERR` 以外のストリームの送信を要求した場合、ストリームが空であっても、少なくとも1つのストリーム型のレコードが送信されます。
<!-- When a role protocol calls for transmitting a stream other than `FCGI_STDERR`, at least one record of the stream type is always transmitted, even if the stream is empty. -->

信頼性の高いプロトコルと単純化されたアプリケーションプログラミングの利益のために、ロールプロトコルは*ほぼリクエストレスポンス*になるように設計されています。真のリクエストレスポンス型プロトコルでは、アプリケーションは最初の出力レコードを送信する前にすべての入力レコードを受信します。リクエストレスポンスプロトコルは、パイプライン化を許可しません。
<!-- Again in the interests of reliable protocols and simplified application programming, role protocols are designed to be *nearly request-response*. In a truly request-response protocol, the application receives all of its input records before sending its first output record. Request-response protocols don't allow pipelining. -->

リクエストレスポンスのルールは、一部の FastCGI ロールには制限が強すぎます。そのため、いくつかのロールプロトコルでは、そのような特定の可能性を許可しています。最初に、アプリケーションは最終的なストリーム入力を除いてすべての入力を受け取ります。アプリケーションが最終的なストリーム入力を受け取り始めると、その出力を書き始めることができます。
<!-- The request-response rule is too restrictive for some FastCGI roles; after all, CGI programs aren't restricted to read all of stdin before starting to write stdout. So some role protocols allow that specific possibility. First the application receives all of its inputs except for a final stream input. As the application begins to receive the final stream input, it can begin writing its output. -->

ロールプロトコルが `FCGI_PARAMS` を用いてCGIプログラムが環境変数から取得する値などのテキスト値を送信する場合、値の長さには終端のヌルバイトは含まれず、値自体にもヌルバイトは含まれない。`environ(7)` 形式の名前と値のペアを提供する必要があるアプリケーションは、名前と値の間に等号を挿入し、値の後にヌルバイトを追加しなければならない。
<!-- When a role protocol uses `FCGI_PARAMS` to transmit textual values, such as the values that CGI programs obtain from environment variables, the length of the value does not include the terminating null byte, and the value itself does not include a null byte. An application that needs to provide `environ(7)` format name-value pairs must insert an equal sign between the name and value and append a null byte after the value. -->

Role プロトコルは CGI のノンパーズドヘッダ機能をサポートしていません。FastCGI アプリケーションは `Status` と `Location` CGI ヘッダを使って応答の状態を設定します。
<!-- Role protocols do not support the non-parsed header feature of CGI. FastCGI applications set response status using the `Status` and `Location` CGI headers. -->

### 6.2. Responder

レスポンダ FastCGI アプリケーションは CGI/1.1 プログラムと同じ目的を持っています。HTTP リクエストに関連するすべての情報を受け取り、HTTP レスポンスを生成します。
<!-- A Responder FastCGI application has the same purpose as a CGI/1.1 program: It receives all the information associated with an HTTP request and generates an HTTP response. -->

レスポンダが CGI/1.1 の各要素をどのようにエミュレートしているかを説明すれば十分です。
<!-- It suffices to explain how each element of CGI/1.1 is emulated by a Responder: -->

  * レスポンダアプリケーションは Web サーバから `FCGI_PARAMS` を介して CGI/1.1 環境変数を受信する。
<!--  * The Responder application receives CGI/1.1 environment variables from the Web server over `FCGI_PARAMS`. -->

  * 次に、レスポンダアプリケーションは Web サーバから `FCGI_STDIN` を介して CGI/1.1 の `stdin` データを受信する。アプリケーションはこのストリームから `CONTENT_LENGTH` バイトを最大で受信してからストリーム終了指示を受信する。(アプリケーションが `CONTENT_LENGTH` バイト未満のバイト数を受け取るのは、HTTP クライアントが `CONTENT_LENGTH` バイトを提供できなかった場合(例えば、クライアントがクラッシュした場合など)のみである。
<!--  * Next the Responder application receives CGI/1.1 `stdin` data from the Web server over `FCGI_STDIN`. The application receives at most `CONTENT_LENGTH` bytes from this stream before receiving the end-of-stream indication. (The application receives less than `CONTENT_LENGTH` bytes only if the HTTP client fails to provide them, e.g. because the client crashed.) -->

  * レスポンダアプリケーションは、CGI/1.1 の `stdout` データを `FCGI_STDOUT` で Web サーバに、CGI/1.1 の `stderr` データを `FCGI_STDERR` で Web サーバに送信します。アプリケーションはこれらのデータを同時に送信します。アプリケーションは、`FCGI_STDOUT` と `FCGI_STDERR` の書き込みを開始する前に `FCGI_PARAMS` の読み込みを終了するのを待たなければなりませんが、これら 2 つのストリームの書き込みを開始する前に `FCGI_STDIN` の読み込みを終了する必要はありません。
<!--  * The Responder application sends CGI/1.1 `stdout` data to the Web server over `FCGI_STDOUT`, and CGI/1.1 `stderr` data over `FCGI_STDERR`. The application sends these concurrently, not one after the other. The application must wait to finish reading `FCGI_PARAMS` before it begins writing `FCGI_STDOUT` and `FCGI_STDERR`, but it needn't finish reading from `FCGI_STDIN` before it begins writing these two streams. -->

  * レスポンダアプリケーションは `stdout` と `stderr` のデータをすべて送信した後、`FCGI_END_REQUEST` レコードを送信する。アプリケーションは `protocolStatus` コンポーネントを `FCGI_REQUEST_COMPLETE` に、`appStatus` コンポーネントを `exit` システムコールで返された CGI プログラムのステータスコードに設定する。
<!--  * After sending all its `stdout` and `stderr` data, the Responder application sends a `FCGI_END_REQUEST` record. The application sets the `protocolStatus` component to `FCGI_REQUEST_COMPLETE` and the `appStatus` component to the status code that the CGI program would have returned via the `exit` system call. -->

`POST` メソッドを実装しているレスポンダは、`FCGI_STDIN` で受信したバイト数と `CONTENT_LENGTH` で受信したバイト数を比較し、2つの値が一致しない場合は更新を中止するべきである。
<!-- A Responder performing an update, e.g. implementing a `POST` method, should compare the number of bytes received on `FCGI_STDIN` with `CONTENT_LENGTH` and abort the update if the two numbers are not equal. -->

### 6.3. Authorizer

認証先 FastCGI アプリケーションは、HTTP 要求に関連付けられたすべての情報を受信し、承認/非承認の決定を生成します。認可された決定の場合、認可者は HTTP 要求に名前と値のペアを関連付けることもできます。認可されていない決定を与える場合、認可者は HTTP クライアントに完全な応答を送信します。
<!-- An Authorizer FastCGI application receives all the information associated with an HTTP request and generates an authorized/unauthorized decision. In case of an authorized decision the Authorizer can also associate name-value pairs with the HTTP request; when giving an unauthorized decision the Authorizer sends a complete response to the HTTP client. -->

CGI/1.1 は HTTP リクエストに関連した情報を表現するための完全に良い方法を定義しているので、オーサライザは同じ表現を使用します。
<!-- Since CGI/1.1 defines a perfectly good way to represent the information associated with an HTTP request, Authorizers use the same representation: -->

  * Authorizer アプリケーションは Web サーバからの HTTP リクエスト情報を `FCGI_PARAMS` ストリームで受け取ります。ウェブサーバは `CONTENT_LENGTH`, `PATH_INFO`, `PATH_TRANSLATED`, `SCRIPT_NAME` ヘッダを送信しません。
<!--  * The Authorizer application receives HTTP request information from the Web server on the `FCGI_PARAMS` stream, in the same format as a Responder. The Web server does not send `CONTENT_LENGTH`, `PATH_INFO`, `PATH_TRANSLATED`, and `SCRIPT_NAME` headers. -->

  * Authorizer アプリケーションはレスポンダと同様に `stdout`, `stderr` のデータを送信する。CGI/1.1 の応答ステータスはリクエストの処理を指定します。アプリケーションがステータス200 (OK) を送信した場合、Webサーバはアクセスを許可します。設定によっては、Web サーバは他の認証元へのリクエストを含む他のアクセスチェックを行うことができます。
<!--  * The Authorizer application sends `stdout` and `stderr` data in the same manner as a Responder. The CGI/1.1 response status specifies the disposition of the request. If the application sends status 200 (OK), the Web server allows access. Depending upon its configuration the Web server may proceed with other access checks, including requests to other Authorizers. -->

    Authorizer アプリケーションのレスポンスには、名前の先頭に `Variable-` が付くヘッダが含まれている場合があります。これらのヘッダは、アプリケーションからウェブサーバに名前と値のペアを通信する。例えば、レスポンスヘッダ
<!--    An Authorizer application's 200 response may include headers whose names are prefixed with `Variable-`. These headers communicate name-value pairs from the application to the Web server. For instance, the response header -->

    ```
    Variable-AUTH_METHOD: database lookup
    ```

    は値 `"database lookup"` に名前 `AUTH-METHOD` を付けて送信する。サーバはこのような名前と値のペアをHTTPリクエストに関連付け、HTTPリクエストを処理する際に実行される後続のCGIやFastCGIリクエストに含めます。アプリケーションが 200 のレスポンスを返すとき、サーバは `Variable-` 接頭辞を持たない名前のレスポンスヘッダを無視し、レスポンスの内容を無視します。

<!--    transmits the value `"database lookup"` with name `AUTH-METHOD`. The server associates such name-value pairs with the HTTP request and includes them in subsequent CGI or FastCGI requests performed in processing the HTTP request. When the application gives a 200 response, the server ignores response headers whose names aren't prefixed with `Variable-` prefix, and ignores any response content. -->

    Authorizer 応答ステータスの値が「200」(OK)以外の場合、Web サーバはアクセスを拒否し、応答ステータス、ヘッダ、およびコンテンツを HTTP クライアントに送り返します。
<!--    For Authorizer response status values other than "200" (OK), the Web server denies access and sends the response status, headers, and content back to the HTTP client. -->

### 6.4. Filter

Filter FastCGI アプリケーションは、HTTP 要求に関連するすべての情報に加えて、Web サーバーに保存されたファイルから追加のデータ ストリームを受け取り、HTTP 応答としてデータ ストリームの「フィルタリングされた」バージョンを生成します。
<!-- A Filter FastCGI application receives all the information associated with an HTTP request, plus an extra stream of data from a file stored on the Web server, and generates a "filtered" version of the data stream as an HTTP response. -->

フィルタは、データファイルをパラメータとするレスポンダと機能的には似ています。データファイル名をパラメータとするレスポンダがデータファイルのアクセス制御を行うのに対し、フィルタはデータファイルとフィルタ自体のアクセス制御を Web サーバーのアクセス制御機構を利用して行うことができます。
<!-- A Filter is similar in functionality to a Responder that takes a data file as a parameter. The difference is that with a Filter, both the data file and the Filter itself can be access controlled using the Web server's access control mechanisms, while a Responder that takes the name of a data file as a parameter must perform its own access control checks on the data file. -->

フィルタのステップはレスポンダのステップと似ています。サーバはフィルタに環境変数を与え、次に標準入力 (通常は `POST` 形式のデータ) を与え、最後にデータファイルの入力を与えます。
<!-- The steps taken by a Filter are similar to those of a Responder. The server presents the Filter with environment variables first, then standard input (normally form `POST` data), finally the data file input: -->

  * レスポンダと同様に、フィルタアプリケーションは Web サーバから `FCGI_PARAMS` で名前と値のペアを受け取る。フィルタアプリケーションはフィルタ固有の変数 `FCGI_DATA_LAST_MOD` と `FCGI_DATA_LENGTH` を受け取る。
<!--  * Like a Responder, the Filter application receives name-value pairs from the Web server over `FCGI_PARAMS`. Filter applications receive two Filter-specific  variables: `FCGI_DATA_LAST_MOD` and `FCGI_DATA_LENGTH`. -->

  * 次にフィルタアプリケーションは、Webサーバから `FCGI_STDIN` を介して CGI/1.1 の `stdin` データを受信します。アプリケーションは、ストリーム終了指示を受信する前に、このストリームから最大で `CONTENT_LENGTH` バイトを受信します (HTTP クライアントが失敗した場合のみ、アプリケーションは `CONTENT_LENGTH` バイト未満のバイトを受信します)。(アプリケーションが `CONTENT_LENGTH` バイト未満のバイト数を受け取るのは、HTTP クライアントが `CONTENT_LENGTH` バイトを提供できなかった場合(例えば、クライアントがクラッシュした場合など)のみである。
<!--  * Next the Filter application receives CGI/1.1 `stdin` data from the Web server over `FCGI_STDIN`. The application receives at most `CONTENT_LENGTH` bytes from this stream before receiving the end-of-stream indication. (The application receives less than `CONTENT_LENGTH` bytes only if the HTTP client fails to provide them, e.g. because the client crashed.) -->

  * 次にフィルタアプリケーションはウェブサーバから `FCGI_DATA` を介してファイルデータを受け取ります。このファイルの最終更新時刻(1970年1月1日(UTC)のエポックからの秒数を整数で表したもの)は `FCGI_DATA_LAST_MOD` であり、アプリケーションはこの変数を参照し、ファイルデータを読まずにキャッシュから応答することができる。アプリケーションはストリーム終了指示を受信するまでに、このストリームから `FCGI_DATA_LENGTH` バイトまでを読み込む。
<!--  * Next the Filter application receives the file data from the Web server over `FCGI_DATA`. This file's last modification time (expressed as an integer number of seconds since the epoch January 1, 1970 UTC) is `FCGI_DATA_LAST_MOD`; the application may consult this variable and respond from a cache without reading the file data. The application reads at most `FCGI_DATA_LENGTH` bytes from this stream before receiving the end-of-stream indication. -->

  * フィルタアプリケーションは、CGI/1.1 の `stdout` データを `FCGI_STDOUT` で Web サーバに、CGI/1.1 の `stderr` データを `FCGI_STDERR` で Web サーバに送信します。アプリケーションはこれらのデータを同時に送信します。アプリケーションは `FCGI_STDOUT` と `FCGI_STDERR` の書き込みを開始する前に `FCGI_STDIN` の読み込みを終了するのを待たなければなりませんが、これら 2 つのストリームの書き込みを開始する前に `FCGI_DATA` の読み込みを終了する必要はありません。
<!--  * The Filter application sends CGI/1.1 `stdout` data to the Web server over `FCGI_STDOUT`, and CGI/1.1 `stderr` data over `FCGI_STDERR`. The application sends these concurrently, not one after the other. The application must wait to finish reading `FCGI_STDIN` before it begins writing `FCGI_STDOUT` and `FCGI_STDERR`, but it needn't finish reading from `FCGI_DATA` before it begins writing these two streams. -->

  * アプリケーションは `stdout` と `stderr` のデータをすべて送信した後、`FCGI_END_REQUEST` レコードを送信します。アプリケーションは `protocolStatus` コンポーネントを `FCGI_REQUEST_COMPLETE` に、`appStatus` コンポーネントを `exit` システムコールで返された同様の CGI プログラムが返すであろうステータスコードに設定します。
<!--  * After sending all its `stdout` and `stderr` data, the application sends a `FCGI_END_REQUEST` record. The application sets the `protocolStatus` component to `FCGI_REQUEST_COMPLETE` and the `appStatus` component to the status code that a similar CGI program would have returned via the `exit` system call. -->

フィルタは、`FCGI_STDIN` で受信したバイト数を `CONTENT_LENGTH` と、`FCGI_DATA` で受信したバイト数を `FCGI_DATA_LENGTH` と比較しなければなりません。数値が一致せず、フィルタがクエリである場合、フィルタのレスポンスはデータがないことを示すものでなければなりません。数値が一致せず、かつフィルタが更新である場合、フィルタは更新を中止しなければならない。
<!-- A Filter should compare the number of bytes received on `FCGI_STDIN` with `CONTENT_LENGTH` and on `FCGI_DATA` with `FCGI_DATA_LENGTH`. If the numbers don't match and the Filter is a query, the Filter response should provide an indication that data is missing. If the numbers don't match and the Filter is an update, the Filter should abort the update. -->

## 7. Errors

FastCGI アプリケーションは、粗雑な形式のガベージコレクションを実行するためなどに、意図的に終了したことを示すために、ステータスがゼロの状態で終了します。ゼロ以外のステータスで終了する FastCGI アプリケーションは、クラッシュしたと見なされます。Web サーバーまたは他のアプリケーション マネージャが、ゼロまたはゼロ以外のステータスで終了するアプリケーショ ンにどのように応答するかは、この仕様の範囲外です。
<!-- A FastCGI application exits with zero status to indicate that it terminated on purpose, e.g. in order to perform a crude form of garbage collection. A FastCGI application that exits with nonzero status is assumed to have crashed. How a Web server or other application manager responds to applications that exit with zero or nonzero status is outside the scope of this specification. -->

Web サーバーは、`SIGTERM` を送信することで FastCGI アプリケーションの終了を要求できます。アプリケーションが `SIGTERM` を無視した場合、ウェブサーバーは `SIGKILL` に頼ることができます。
<!-- A Web server can request that a FastCGI application exit by sending it `SIGTERM`. If the application ignores `SIGTERM` the Web server can resort to `SIGKILL`. -->

FastCGI アプリケーションはアプリケーションレベルのエラーを `FCGI_STDERR` ストリームと `FCGI_END_REQUEST` レコードの `appStatus` コンポーネントで報告します。多くの場合、エラーは `FCGI_STDOUT` ストリームを介して直接ユーザーに報告されます。
<!-- FastCGI applications report application-level errors with the `FCGI_STDERR` stream and the `appStatus` component of the `FCGI_END_REQUEST` record. In many cases an error will be reported directly to the user via the `FCGI_STDOUT` stream. -->

Unixでは、アプリケーションはFastCGIプロトコルエラーやFastCGI環境変数の構文エラーなどの下位レベルのエラーを `syslog`に報告します。エラーの深刻度に応じて、アプリケーションは継続するか、ゼロ以外のステータスで終了します。
<!-- On Unix, applications report lower-level errors, including FastCGI protocol errors and syntax errors in FastCGI environment variables, to `syslog`. Depending upon the severity of the error, the application may either continue or exit with nonzero status. -->

## 8. Types and constants

```c
/*
 * Listening socket file number
 */
#define FCGI_LISTENSOCK_FILENO 0

typedef struct {
    unsigned char version;
    unsigned char type;
    unsigned char requestIdB1;
    unsigned char requestIdB0;
    unsigned char contentLengthB1;
    unsigned char contentLengthB0;
    unsigned char paddingLength;
    unsigned char reserved;
} FCGI_Header;

/*
 * Number of bytes in a FCGI_Header.  Future versions of the protocol
 * will not reduce this number.
 */
#define FCGI_HEADER_LEN  8

/*
 * Value for version component of FCGI_Header
 */
#define FCGI_VERSION_1           1

/*
 * Values for type component of FCGI_Header
 */
#define FCGI_BEGIN_REQUEST       1
#define FCGI_ABORT_REQUEST       2
#define FCGI_END_REQUEST         3
#define FCGI_PARAMS              4
#define FCGI_STDIN               5
#define FCGI_STDOUT              6
#define FCGI_STDERR              7
#define FCGI_DATA                8
#define FCGI_GET_VALUES          9
#define FCGI_GET_VALUES_RESULT  10
#define FCGI_UNKNOWN_TYPE       11
#define FCGI_MAXTYPE (FCGI_UNKNOWN_TYPE)

/*
 * Value for requestId component of FCGI_Header
 */
#define FCGI_NULL_REQUEST_ID     0

typedef struct {
    unsigned char roleB1;
    unsigned char roleB0;
    unsigned char flags;
    unsigned char reserved[5];
} FCGI_BeginRequestBody;

typedef struct {
    FCGI_Header header;
    FCGI_BeginRequestBody body;
} FCGI_BeginRequestRecord;

/*
 * Mask for flags component of FCGI_BeginRequestBody
 */
#define FCGI_KEEP_CONN  1

/*
 * Values for role component of FCGI_BeginRequestBody
 */
#define FCGI_RESPONDER  1
#define FCGI_AUTHORIZER 2
#define FCGI_FILTER     3

typedef struct {
    unsigned char appStatusB3;
    unsigned char appStatusB2;
    unsigned char appStatusB1;
    unsigned char appStatusB0;
    unsigned char protocolStatus;
    unsigned char reserved[3];
} FCGI_EndRequestBody;

typedef struct {
    FCGI_Header header;
    FCGI_EndRequestBody body;
} FCGI_EndRequestRecord;

/*
 * Values for protocolStatus component of FCGI_EndRequestBody
 */
#define FCGI_REQUEST_COMPLETE 0
#define FCGI_CANT_MPX_CONN    1
#define FCGI_OVERLOADED       2
#define FCGI_UNKNOWN_ROLE     3

/*
 * Variable names for FCGI_GET_VALUES / FCGI_GET_VALUES_RESULT records
 */
#define FCGI_MAX_CONNS  "FCGI_MAX_CONNS"
#define FCGI_MAX_REQS   "FCGI_MAX_REQS"
#define FCGI_MPXS_CONNS "FCGI_MPXS_CONNS"

typedef struct {
    unsigned char type;    
    unsigned char reserved[7];
} FCGI_UnknownTypeBody;

typedef struct {
    FCGI_Header header;
    FCGI_UnknownTypeBody body;
} FCGI_UnknownTypeRecord;
```

## 9. References

National Center for Supercomputer Applications, [The Common Gateway Interface](http://hoohoo.ncsa.illinois.edu/cgi/), version CGI/1.1.

D.R.T. Robinson, [The WWW Common Gateway Interface Version 1.1](http://cgi-spec.golux.com/), Internet-Draft, 15 February 1996.

### Updated references

W3C, [CGI: Common Gateway Interface](https://www.w3.org/CGI/)

Robinson, D. and K. Coar, [The Common Gateway Interface (CGI) Version 1.1](https://tools.ietf.org/html/rfc3875), RFC 3875, DOI 10.17487/RFC3875, October 2004

## A. Table: Properties of the record types

次の図は、すべてのレコードタイプをリストアップし、それぞれのこれらのプロパティを示しています。
<!-- The following chart lists all of the record types and indicates these properties of each: -->

  * このタイプのレコードは、Webサーバからアプリケーションにのみ送信できます。他のタイプのレコードは、アプリケーションからWebサーバーにのみ送信できます。
<!--  * **WS->App:** records of this type can only be sent by the Web server to the application. Records of other types can only be sent by the application to the Web server. -->

  * このタイプのレコードには、Webサーバーのリクエストに固有ではない情報が含まれており、NULLリクエストIDを使用します。他のタイプのレコードには、リクエスト固有の情報が含まれており、NULLリクエストIDを使用することはできません。
<!--  * **management:** records of this type contain information that is not specific to a Web server request, and use the null request ID. Records of other types contain request-specific information, and cannot use the null request ID. -->

  * このタイプのレコードはストリームを形成し、空の `contentData` を持つレコードで終了します。他のタイプのレコードは離散的で、それぞれが意味のあるデータ単位を持ちます。
<!--  * **stream:** records of this type form a stream, terminated by a record with empty `contentData`. Records of other types are discrete; each carries a meaningful unit of data. -->

|                          | WS->App | management | stream  |
|--------------------------|---------|------------|---------|
| `FCGI_GET_VALUES`        |    x    |     x      |         |
| `FCGI_GET_VALUES_RESULT` |         |     x      |         |
| `FCGI_UNKNOWN_TYPE`      |         |     x      |         |
| `FCGI_BEGIN_REQUEST`     |    x    |            |         |
| `FCGI_ABORT_REQUEST`     |    x    |            |         |
| `FCGI_END_REQUEST`       |         |            |         |
| `FCGI_PARAMS`            |    x    |            |    x    |
| `FCGI_STDIN`             |    x    |            |    x    |
| `FCGI_DATA`              |    x    |            |    x    |
| `FCGI_STDOUT`            |         |            |    x    |
| `FCGI_STDERR`            |         |            |    x    |

## B. Typical protocol message flow

例のための追加の表記法。
<!-- Additional notational conventions for the examples: -->

  * ストリームレコード(`FCGI_PARAMS`, `FCGI_STDIN`, `FCGI_STDOUT`, `FCGI_STDERR`)の `contentData` は文字列として表現されます。で終わる文字列は、`" ... "` で終わる文字列は長すぎて表示できないので、プレフィックスのみを表示します。  
  * Webサーバに送信されたメッセージは、Webサーバから受信したメッセージに対してインデントされます。
  * メッセージは、アプリケーションが経験した時系列で表示されます。

<!--  * The `contentData` of stream records (`FCGI_PARAMS`, `FCGI_STDIN`, `FCGI_STDOUT`, and `FCGI_STDERR`) is represented as a character string. A string ending in `" ... "` is too long to display, so only a prefix is shown.
  * Messages sent to the Web server are indented with respect to messages received from the Web server.
  * Messages are shown in the time sequence experienced by the application. -->

1\. 単純なリクエストで `stdin` のデータがない場合、応答は成功します。
<!-- 1\. A simple request with no data on `stdin`, and a successful response: -->

```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, 0}}
{FCGI_PARAMS,          1,
    "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_STDIN,           1, ""}

    {FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n<html>\n<head> ... "}
    {FCGI_STDOUT,      1, ""}
    {FCGI_END_REQUEST, 1, {0, FCGI_REQUEST_COMPLETE}}
```

2\. 例1と似ていますが、今回は `stdin` のデータを使用しています。ウェブサーバは、以前よりも多くの `FCGI_PARAMS` レコードを使ってパラメータを送信することを選択しています。
<!-- 2\. Similar to example 1, but this time with data on `stdin`. The Web server chooses to send the parameters using more `FCGI_PARAMS` records than before: -->

```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, 0}}
{FCGI_PARAMS,          1, "\013\002SERVER_PORT80\013\016SER"}
{FCGI_PARAMS,          1, "VER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_STDIN,           1, "quantity=100&item=3047936"}
{FCGI_STDIN,           1, ""}

    {FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n<html>\n<head> ... "}
    {FCGI_STDOUT,      1, ""}
    {FCGI_END_REQUEST, 1, {0, FCGI_REQUEST_COMPLETE}}
```

3\. 例1と似ていますが、今回はアプリケーションがエラーを検出します。アプリケーションはメッセージを `stderr` にログに記録し、クライアントにページを返し、ウェブサーバにゼロではない終了ステータスを返します。アプリケーションは、より多くの `FCGI_STDOUT` レコードを使ってページを送信することを選択します。
<!-- 3\. Similar to example 1, but this time the application detects an error. The application logs a message to `stderr`, returns a page to the client, and returns non-zero exit status to the Web server. The application chooses to send the page using more `FCGI_STDOUT` records: -->

```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, 0}}
{FCGI_PARAMS,          1,
    "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_STDIN,           1, ""}

    {FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n<ht"}
    {FCGI_STDERR,      1, "config error: missing SI_UID\n"}
    {FCGI_STDOUT,      1, "ml>\n<head> ... "}
    {FCGI_STDOUT,      1, ""}
    {FCGI_STDERR,      1, ""}
    {FCGI_END_REQUEST, 1, {938, FCGI_REQUEST_COMPLETE}}
```

4\. 1つの接続に多重化された例1の2つのインスタンス。最初のリクエストは2番目よりも難しいので、アプリケーションは順不同でリクエストを終了します。
<!-- 4\. Two instances of example 1, multiplexed onto a single connection. The first request is more difficult than the second, so the application finishes the requests out of order: -->

```c
{FCGI_BEGIN_REQUEST,   1, {FCGI_RESPONDER, FCGI_KEEP_CONN}}
{FCGI_PARAMS,          1,
    "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_PARAMS,          1, ""}
{FCGI_BEGIN_REQUEST,   2, {FCGI_RESPONDER, FCGI_KEEP_CONN}}
{FCGI_PARAMS,          2,
    "\013\002SERVER_PORT80\013\016SERVER_ADDR199.170.183.42 ... "}
{FCGI_STDIN,           1, ""}

    {FCGI_STDOUT,      1, "Content-type: text/html\r\n\r\n"}

{FCGI_PARAMS,          2, ""}
{FCGI_STDIN,           2, ""}

    {FCGI_STDOUT,      2, "Content-type: text/html\r\n\r\n<html>\n<head> ... "}
    {FCGI_STDOUT,      2, ""}
    {FCGI_END_REQUEST, 2, {0, FCGI_REQUEST_COMPLETE}}
    {FCGI_STDOUT,      1, "<html>\n<head> ... "}
    {FCGI_STDOUT,      1, ""}
    {FCGI_END_REQUEST, 1, {0, FCGI_REQUEST_COMPLETE}}
```

* * *

*&copy; Copyright 1995, 1996 Open Market, Inc. / mbrown@openmarket.com*
