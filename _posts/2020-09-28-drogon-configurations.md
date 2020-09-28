---
layout: post
excerpt_separator: <!--more-->
title: Drogon - Configurations
categories: drogon c++
---

# 드로곤(Drogon) - 설정 항목
## 개요

드로곤은 애플리케이션의 각종 설정 항목들을 JSON 을 통해 정의하고 참조한다. 이 문서는 드로곤의 설정 항목들에 대해서 살펴본다.
<!--more-->

## 설정 항목

현재 시점에서 설정 파일 항목들은 다음과 같다. 아래와 같이 주석이 달려 있는데, JSON 에서 주석은 유효한 표현이 아니다. 따라서 실제로 사용할때는 제거하도록 한다.
처음 프로젝트 생성할 때는 JSON 파일을 사용하지 않고, `main.cc` 내부에서 주석 처리되어 있다. 필요하다면 주석을 해제하면 된다.

```
/* This is a JSON format configuration file
 */
{
    /*
    //ssl:The global ssl files setting
    //ssl:전역 ssl 파일 설정
    "ssl": {
        "cert": "../../trantor/trantor/tests/server.pem",
        "key": "../../trantor/trantor/tests/server.pem"
    },
    "listeners": [
        {
            //address: Ip address,0.0.0.0 by default
            //주소: Ip 주소, 0.0.0.0 이 기본
            "address": "0.0.0.0",
            //port: Port number
            //포트: 포트 번호
            "port": 80,
            //https: If true, use https for security,false by default
            //https: 참이면 https 를 사용한다. false 가 기본
            "https": false
        },
        {
            "address": "0.0.0.0",
            "port": 443,
            "https": true,
            //cert,key: Cert file path and key file path, empty by default,
            //if empty, use the global setting
            //cert, key: Cert 파일 경로 및 키 파일 경로, 빈 값이 기본
            //비어 있다면 전역 설정을 사용한다.
            "cert": "",
            "key": ""
        }
    ],
    "db_clients": [
        {
            //name: Name of the client,'default' by default
            //이름: 클라이언트 이름 default 가 기본
            //"name":"",
            //rdbms: Server type, postgresql,mysql or sqlite3, "postgresql" by default
            //rdbms: 서버 유형, postgresql,mysql 또는 sqlite3, "postgresql" 가 기본
            "rdbms": "postgresql",
            //filename: Sqlite3 db file name
            //filename: Sqlite3 db 파일명
            //"filename":"",
            //host: Server address,localhost by default
            //host: 서버 주소, localhost 가 기본
            "host": "127.0.0.1",
            //port: Server port, 5432 by default
            //port: 서버 포트, 5432 가 기본
            "port": 5432,
            //dbname: Database name
            //dbname: 데이터베이스 이름
            "dbname": "test",
            //user: 'postgres' by default
            //user: 'postgres' 가 기본
            "user": "",
            //passwd: '' by default
            //passwd: '' 가 기본
            "passwd": "",
            //is_fast: false by default, if it is true, the client is faster but user can't call
            //any synchronous interface of it.
            //is_fast: false 가 기본, 이 값이 참이면 client 는 더 빨라지지만 동기 인터페이스를 사용할 수 없다
            "is_fast": false,
            //client_encoding: The character set used by the client. it is empty string by default which
            //means use the default character set.
            //client_encoding: 클라이언트에서 사용하는 문자셋. 빈 문자열이 기본인데 이는 기본 문자셋을 의미한다.
            //"client_encoding": "",
            //connection_number: 1 by default, if the 'is_fast' is true, the number is the number of
            //connections per IO thread, otherwise it is the total number of all connections.
            //connection_number: 1 이 기본이다. is_fast 가 참이면 이 숫자는 IO 스레드당 커넥션의 수이며, 그렇지 않다면 모든 커넥션의 수이다.
            "connection_number": 1
        }
    ],*/
    "app": {
        //threads_num: The number of IO threads, 1 by default, if the value is set to 0, the number of threads
        //is the number of CPU cores
        //threads_num: IO 스레드의 수, 1이 기본이다. 0으로 설정하면 스레드 수는 CPU 코어 수가 된다.
        "threads_num": 1,
        //enable_session: False by default
        //enable_session: false 가 기본
        "enable_session": false,
        "session_timeout": 0,
        //document_root: Root path of HTTP document, defaut path is ./
        //document_root: HTTP 도큐머트의 루트 경로. 기본은 ./ 이다
        "document_root": "./",
        //home_page: Set the HTML file of the home page, the default value is "index.html"
        //If there isn't any handler registered to the path "/", the home page file in the "document_root" is send to clients as a response
        //to the request for "/".
        //home_page: 홈 페이지 HTML 파일을 설정한다. 기본은 index.html 이다.
        // "/" 경로로 등록된 핸들러가 존재하지 않는다면 "/" 요청에 대한 응답으로 "document_root" 에 있는 홈 페이지 파일이 클라이언트에 전송된다.
        "home_page": "index.html",
        //static_file_headers: Headers for static files
        //static_file_headers: 정적 파일에 대한 헤더
        /*"static_file_headers": [
          {
                "name": "field-name",
                "value": "field-value"
          }
        ],*/
        //upload_path: The path to save the uploaded file. "uploads" by default.
        //If the path isn't prefixed with /, ./ or ../,
        //it is relative path of document_root path
        //upload_path: 업로드된 파일을 저장할 경로. "uploads" 가 기본이다.
        // 경로 앞에 ./, ../ 가 붙지 않는다면 document_root 의 상대 경로가 된다.
        "upload_path": "uploads",
        /* file_types:
         * HTTP download file types,The file types supported by drogon
         * by default are "html", "js", "css", "xml", "xsl", "txt", "svg",
         * "ttf", "otf", "woff2", "woff" , "eot", "png", "jpg", "jpeg",
         * "gif", "bmp", "ico", "icns", etc. */
        "file_types": [
            "gif",
            "png",
            "jpg",
            "js",
            "css",
            "html",
            "ico",
            "swf",
            "xap",
            "apk",
            "cur",
            "xml"
        ],
        //locations: An array of locations of static files for GET requests.
        //locations: GET 요청에 대한 정적 파일의 경로 배열
        "locations": [{
            //uri_prefix: The URI prefix of the location prefixed with "/", the default value is "" that disables the location.
            //uri_prefix: 앞에 "/" 가 붙은 경로의 URI 전치, 기본값은 "" 이며, 경로를 사용하지 못한다.
            //"uri_prefix": "/.well-known/acme-challenge/",
            //default_content_type: The default content type of the static files without
            //an extension. empty string by default.
            //default_content_type: 확장자가 없는 정적 파일의 기본 컨텐트 유형이다. 빈 값이 기본
            "default_content_type": "text/plain",
            //alias: The location in file system, if it is prefixed with "/", it
            //presents an absolute path, otherwise it presents a relative path to
            //the document_root path.
            //The default value is "" which means use the document root path as the location base path.
            //alias: 파일 시스템에서의 위치. / 로 시작하면 절대 경로를 의미하고, 그렇지 않으면 document_root 경로에 대한 상대 경로를 의미한다.
            //기본값은 "" 이며 document root 경로를 기본 경로로 사용함을 의미한다.
            "alias": "",
            //is_case_sensitive: indicates whether the URI prefix is case sensitive.
            //is_case_sensitive: URI 전치가 대소문자 구분인지 나태낸다.
            "is_case_sensitive": false,
            //allow_all: true by default. If it is set to false, only static files with a valid extension can be accessed.
            //allow_all: ture 가 기본. false 로 설정되면 유효한 확장자를 가진 정적 파일만 접근할 수 있다.
            "allow_all": true,
            //is_recursive: true by default. If it is set to false, files in sub directories can't be accessed.
            //is_recursive: ture 가 기본. false 로 설정하면 하위 디렉터리에 있는 파일은 접근할 수 없다.
            "is_recursive": true,
            //filters: string array, the filters applied to the location.
            //filters: 문자열 배열. 경로에 적용할 필터들.
            "filters": []
        }],
        //max_connections: maximum connections number,100000 by default
        //max_connections: 최대 커넥션 수. 100000 이 기본이다
        "max_connections": 100000,
        //max_connections_per_ip: maximum connections number per clinet,0 by default which means no limit
        //max_connections_per_ip: 클라이언트당 최대 커텍션 수. 0 이 기본이며 제한이 없을 의미한다.
        "max_connections_per_ip": 0,
        //Load_dynamic_views: False by default, when set to true, drogon
        //compiles and loads dynamically "CSP View Files" in directories defined
        //by "dynamic_views_path"
        //load_dynamic_views: false 가 기본이다. true 로 설정하면 drogon 은 dynamic_views_path 에 정의된 디렉터리에 있는
        // CSP 뷰 파일을 동적으로 컴파일하고 로드힌다.
        "load_dynamic_views": false,
        //dynamic_views_path: If the path isn't prefixed with /, ./ or ../,
        //it is relative path of document_root path
        //dynamic_views_path: 경로에 /, ./ 또는 ../ 가 붙어 있다면 document_root 경로에 대한 상대 경로가 된다
        "dynamic_views_path": [
            "./views"
        ],
        //dynamic_views_output_path: Default by an empty string which means the output path of source
        //files is the path where the csp files locate. If the path isn't prefixed with /, it is relative
        //path of the current working directory.
        //dynamic_views_output_path: 기본 값은 빈 문자열이며 소스 파일의 출력 경로는 csp 파일이 위치하는 경로임을 의미한다. 경로가 / 로 시작하지 안흐면
        //경로는 현재 작업 디렉터리에 대한 상대 경로가 된다.
        "dynamic_views_output_path": "",
        //enable_unicode_escaping_in_json: true by default, enable unicode escaping in json.
        //enable_unicode_escaping_in_json: true 기본이다. 유니코드 이스케이핑을 활성화한다.
        "enable_unicode_escaping_in_json": true,
        //log: Set log output, drogon output logs to stdout by default
        //log: log 출력을 설정한다. drogon 은 기본적으로 log 를 stdout 으로 출력한다.
        "log": {
            //log_path: Log file path,empty by default,in which case,logs are output to the stdout
            //log_path: 로그 파일 경로, 빈 값이 기본이며 이 경우 로그는 stdout 으로 출력된다.
            //"log_path": "./",
            //logfile_base_name: Log file base name,empty by default which means drogon names logfile as
            //drogon.log ...
            //logfile_base_name: 로그 파일 기본 이름. 빈 값이 기본이며 로그파일 이름을 drogon.log 로 사용함을 의미한다
            "logfile_base_name": "",
            //log_size_limit: 100000000 bytes by default,
            //When the log file size reaches "log_size_limit", the log file is switched.
            //log_size_limit: 100000000 가 기본. 약 100M
            // 로그 파일이 log_size_limit 에 도달하면 로그 파일이 전환된다.
            "log_size_limit": 100000000,
            //log_level: "DEBUG" by default,options:"TRACE","DEBUG","INFO","WARN"
            //The TRACE level is only valid when built in DEBUG mode.
            //log_level: "DEBUG" 가 기본. "TRACE","DEBUG","INFO","WARN" 를 사용할 수 있다.
            // TRACE 수준은 DEBUG 모드로 빌드했을때만 유효하다.
            "log_level": "DEBUG"
        },
        //run_as_daemon: False by default
        //run_as_daemon: false 가 기본
        "run_as_daemon": false,
        //relaunch_on_error: False by default, if true, the program will be restart by the parent after exiting;
        //relaunch_on_error: false 가 기본. true 이면 프로그램 종료시 부모에 의해 재시작 된다.
        "relaunch_on_error": false,
        //use_sendfile: True by default, if true, the program
        //uses sendfile() system-call to send static files to clients;
        //use_sendfile: true 가 기본. true 이면 프로그램은 클라이언트로 정적 파일을 전송하기 위해 sendfile() 시스템 호출을 사용한다.
        "use_sendfile": true,
        //use_gzip: True by default, use gzip to compress the response body's content;
        //use_gzip: true 가 기본이다. 응답 바디를 압축하기 위해 gzip 을 사용한다.
        "use_gzip": true,
        //use_brotli: False by default, use brotli to compress the response body's content;
        //use_brotli: false 가 기본. 응답 바디를 압축하기 위해 brotli 를 사용한다.
        "use_brotli": false,
        //static_files_cache_time: 5 (seconds) by default, the time in which the static file response is cached,
        //0 means cache forever, the negative value means no cache
        //static_files_cache_time: 5 (초)가 기본. 정적 파일 응답이 캐싱되는 시간이다. 0은 무한히 캐싱함을 의미한다. 음수는 캐싱하지 않음을 의미한다.
        "static_files_cache_time": 5,
        //simple_controllers_map: Used to configure mapping from path to simple controller
        //simple_controllers_map: 경로를 심플 컨트롤러에 맵핑하기 위해 사용한다
        "simple_controllers_map": [{
            "path": "/path/name",
            "controller": "controllerClassName",
            "http_methods": [
                "get",
                "post"
            ],
            "filters": [
                "FilterClassName"
            ]
        }],
        //idle_connection_timeout: Defaults to 60 seconds, the lifetime
        //of the connection without read or write
        //idle_connection_timeout: 기본은 60 초. 읽기나 쓰기가 없는 커넥션의 생존시간이다
        "idle_connection_timeout": 60,
        //server_header_field: Set the 'Server' header field in each response sent by drogon,
        //empty string by default with which the 'Server' header field is set to "Server: drogon/version string\r\n"
        //server_header_field: drogon 에 의해 전송된 헤더에 Server 헤더를 추가한다.
        //빈 값이 기본이며 "Server: drogon/version string\r\n" 로 설정하게 된다.
        "server_header_field": "",
        //enable_server_header: Set true to force drogon to add a 'Server' header to each HTTP response. The default
        //value is true.
        //enable_server_header: true 를 설정하면 drogon 이 Server 헤더를 HTTP 응답 헤더에 추가하게 된다. 기본은 true 이다.
        "enable_server_header": true,
        //enable_date_header: Set true to force drogon to add a 'Date' header to each HTTP response. The default
        //value is true.
        //enable_date_header: true 로 설정하면 Date 헤더를 HTTP 응답에 추가하게 된다. 기본은 true 이다.
        "enable_date_header": true,
        //keepalive_requests: Set the maximum number of requests that can be served through one keep-alive connection.
        //After the maximum number of requests are made, the connection is closed.
        //The default value of 0 means no limit.
        //keepalive_requests: 하나의 keep-alive 커넥션을 통해 제공될 수 있는 최대 요청 수를 설정한다.
        //최대 수를 넘어선 이후에 커넥션은 닫힌다.
        //기본 값은 0이며 제한이 없음을 나타낸다.
        "keepalive_requests": 0,
        //pipelining_requests: Set the maximum number of unhandled requests that can be cached in pipelining buffer.
        //After the maximum number of requests are made, the connection is closed.
        //The default value of 0 means no limit.
        //pipelining_requests: 파이프라이닝 버퍼에서 캐싱될 수 있는 처리되지 않는 최대 요청수를 설정한다.
        //최대 요청수가 넘어선 이후에 커넥션은 닫힌다.
        //기본값은 0이며 제한이 없음을 의미한다
        "pipelining_requests": 0,
        //gzip_static: If it is set to true, when the client requests a static file, drogon first finds the compressed
        //file with the extension ".gz" in the same path and send the compressed file to the client.
        //The default value of gzip_static is true.
        //gzip_static: true 로 설정하면 클라이언트가 정적 파일을 요청했을 때 drogon 은 동일 경로에서 .gz 확장자로 압출된 파일을 먼저 찾아 클라이언트에 전송한다.
        //기본 값은 true 이다.
        "gzip_static": true,
        //br_static: If it is set to true, when the client requests a static file, drogon first finds the compressed
        //file with the extension ".br" in the same path and send the compressed file to the client.
        //The default value of br_static is true.
        //br_static: true 로 설정하면 클라이언트가 정적 파일을 요청했을 떼 drogon 은 동일 경로에서 .br 확장자로 압축된 파일을 찾아 클라이언트에 전송한다.
        //기본값은 true 이다
        "br_static": true,
        //client_max_body_size: Set the maximum body size of HTTP requests received by drogon. The default value is "1M".
        //One can set it to "1024", "1k", "10M", "1G", etc. Setting it to "" means no limit.
        //client_max_body_size: drogon 이 받을 수 있는 HTTP 요청 바디 최대 크기를 설정한다. 기본 값은 "1M" 이다.
        //"1024", "1k", "10M", "1G" 등과 같이 설정할 수 있다. "" 로 설정하면 제한이 없음을 의미한다.
        "client_max_body_size": "1M",
        //max_memory_body_size: Set the maximum body size in memory of HTTP requests received by drogon. The default value is "64K" bytes.
        //If the body size of a HTTP request exceeds this limit, the body is stored to a temporary file for processing.
        //Setting it to "" means no limit.
        //max_memory_body_size: drogon 이 메모리에 받을 수 있는 HTTP 요청 최대 크기를 설정한다. 기본값은 64K 이다. HTTP 요청의 바디 크기가 이 값을
        //초과하면 바디는 임시파일에 저장되어 처리된다.
        "client_max_memory_body_size": "64K",
        //client_max_websocket_message_size: Set the maximum size of messages sent by WebSocket client. The default value is "128K".
        //One can set it to "1024", "1k", "10M", "1G", etc. Setting it to "" means no limit.
        //client_max_websocket_message_size: 웹소켓 클라인언트에서 보낼 수 있는 메시지의 최대 크기를 설정한다. 기본 값은 128K 이다
        //"1024", "1k", "10M", "1G" 등과 같이 설정할 수 있으며, "" 은 제한이 없음을 의미한다.
        "client_max_websocket_message_size": "128K"
    },
    //plugins: Define all plugins running in the application
    //plugins: 애플리케이션에서 실핼되는 모든 플러그인을 정의한다.
    "plugins": [{
        //name: The class name of the plugin
        //name: 플러그인의 클래스 이름
        //"name": "drogon::plugin::SecureSSLRedirector",
        //dependencies: Plugins that the plugin depends on. It can be commented out
        //dependencies: 플로그인이 의존하는 플러그인. 주석처리 할 수 있음.
        "dependencies": [],
        //config: The configuration of the plugin. This json object is the parameter to initialize the plugin.
        //It can be commented out
        //config: 플러그인의 설정. 플러그인을 초기화하기 위해 사용되는 매개변수이다.
        "config": {
            "ssl_redirect_exempt": [".*\\.jpg"],
            "secure_ssl_host": "localhost:8849"
        }
    }],
    //custom_config: custom configuration for users. This object can be get by the app().getCustomConfig() method.
    //custom_config: 커스텀 설정. app().getCustomConfig() 메소드를 통해 접근할 수 있다.
    "custom_config": {}
}
```

# 마무리

설정 파일을 보면, 로깅, 필터, 플러그인, 파일 업로드, SSL 지원, gzip 압축 등 기본적으로 비즈니스 로직을 처리하는 데 무리가 없을 것으로 보인다. MySQL 액세스가 어떻게 지원되는지 
궁금한데 이부분은 후속 포스트에서 천천히 살펴보기로 한다. 