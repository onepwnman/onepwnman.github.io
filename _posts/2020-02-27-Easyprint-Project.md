---
layout: post
title: "Easyprint, 3D Printing Web Interface"
tags: [Project, Web, Flask, 3D Printer]
reference: 
  - "프로젝트 링크" : https://github.com/onepwnman/easyprint



eventlet: https://eventlet.net/
CuraEngine: https://github.com/Ultimaker/CuraEngine
Marlin: https://marlinfw.org/
octoprint: https://octoprint.org/
mjpg-streamer: https://github.com/jacksonliam/mjpg-streamer
gcode: https://reprap.org/wiki/G-code

---

<!--
## Table of contents
> * &nbsp;[개발 시작 계기](#개발-시작-계기)
> * &nbsp;[구성요소, 사용한 스킬 셋](#구성요소-사용한-스킬-셋)
> * &nbsp;[서버 사이드 구조](#서버-사이드-구조)
> * &nbsp;[프로젝트 결과](#프로젝트-결과)
> * &nbsp;[어려웠던 점, 어떻게 극복했나](#어려웠던-점-어떻게-극복했나)
> * &nbsp;[프로젝트의 한계, 개선방안](#프로젝트의-한계-개선방안)
> * &nbsp;[기타 느낀 점](#기타-느낀-점)
       
<br>
<br>
<br>

---  

<br>
<br>
-->
### 개발 시작 계기 
3d 프린팅에 관심이 많아서 저가형 3d 프린터를 구매.  하지만 출력 방법이 생각보다 너무 번거로움.  
프린팅 베드의 수평 레벨이 조금만 어긋나도 처음부터 다시 출력해야 하는 일이 비일비재.   
또한 한번 프린팅을 하려면 슬라이서 프로그램으로 슬라이싱을 해서 마이크로sd카드에 넣고 마이크로sd카드를 3d 프린터에 끼우고 프린팅을 해야 하는 번거로운 과정을 거침 따라서 이를 자동화 할 수 있는 방법을 찾기 시작.  

[octoprint](https://octoprint.org/)라는 원격으로 3d 프린팅을 가능케 해주는 웹 인터페이스가 있음 하지만 여전히 슬라이싱을 따로 해서 [octoprint](https://octoprint.org/)에 파일을 올려야 한다.   
따라서 모든 것을 한데 묶어서 하나의 웹 인터페이스에서 슬라이싱 부터 프린팅까지 전부 할 수 없을까?  해서 프로젝트를 시작하게 되었음.  


<br>
<br>

### 구성요소, 사용한 스킬 셋

서버 환경:&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;Raspberry Pi위에 Docker Container  
메인 웹서버:&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[eventlet](https://eventlet.net/)을 이용한 비동기 flask 웹서버   
http & https 프록시 서버: &emsp;&emsp;Nginx  
작업큐:&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;[Redis Queue(rq)](https://python-rq.org/)         
서버와 클라이언트와의 통신:&emsp; Websocket   
데이터베이스와 ORM:&nbsp;&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;MYSQL와 SQLAlchemy(flask-sqlalchmey)    
메일서비스:&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;flask-mail   
메일 인증:&nbsp;&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;JWT 토큰   
클라이언트 UI :&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;Bootstrap   
배포와 스크립팅:&nbsp;&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp; Docker, Shell Script   
슬라이싱 엔진:&nbsp;&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; [CuraEngine]({{ page.CuraEngine }})              
웹캠서버:&nbsp;&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; [mjpg-streamer]({{ page.mjpg-streamer }})        
3D Printer:&nbsp;&nbsp;&nbsp;&nbsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp; [anet a8 프린터](https://3dprint.wiki/reprap/anet/a8)와 [Marlin 펌웨어]({{ page.Malin }})       

그 밖의 javascript 라이브러리들 three.js, pnotify, dropzone, selectable과 다양한 flask 모듈들(데이터베이스 마이그레이션을 
위한 flask-migrate, 폼 관련 모듈 flask-wtf, 로그인 관련 모듈 flask-login 등등..)

<br>
<br>

### 서버 사이드 구조
<br>
![diagram](/assets/images/easyprint-project/diagram.jpg "Overview"){: width="1200" height="600"}
    
    

<br>
> * &nbsp;물리 서버 환경은 라즈베리 파이 위의 하나의 도커 컨테이너에서 모든 서비스들이 돌아가게 설계하였음.
> * &nbsp;Flask와 [eventlet]({{ page.eventlet }})을 이용해 비동기 웹서버를 구현하였음.       
> * &nbsp;[eventlet]({{ page.eventlet }})은 ssl과 같이 사용하였을 때 issue가 있음으로 ssl 프록시 서버로 nginx를 두었음.   
> * &nbsp;데이터베이스 안에는 크게 User, Task, Print 세 개의 테이블을 두었으며 ORM으로 SQLAlchemy를 사용하였음.   
> * &nbsp;각각의 모델링 파일들과 [gcode]({{ page.gcode }}) 파일명은 해시값을 사용하여 다른 사용자의 파일을 유추하여 프린트하지 못하도록 보안을 강화하였음.      
> * &nbsp;태스크는 크게 Sending Mail, Printing, Slicing등이 있으며 각각의 테스크는 [Redis Queue(rq)](https://python-rq.org/)에 저장되어 비동기적으로 실행하였음.     
> * &nbsp;슬라이싱 요청, 프린팅 요청, 프린팅 완료 알림과 같은 각각의 이벤트들을 Websocket으로 데이터를 주고받도록 설계하였음.    
> * &nbsp;슬라이싱은 [CuraEngine]({{ page.CuraEngine }})으로 프린팅은 [Marlin]({{ page.Marlin }})과의 시리얼 통신으로 구현하였음.       
> * &nbsp;회원가입, 비밀번호 찾기, 회원탈퇴, 비밀번호 변경 등 보안과 관련된 작업들은 JWT토큰을 이용하여 메일인증을 거치도록 하였음.         
> * &nbsp;웹캠은 [mjpg-streamer]({{ page.mjpg-streamer }}) 서버에서 별도로 처리하였음.         



<br>
<br>
<br>
<br>

### 프로젝트 결과 
<br>
#### 프린팅 데모 영상

[![Video Label](http://img.youtube.com/vi/isiyjE_qeqg/0.jpg)](https://youtu.be/isiyjE_qeqg)
<br>


#### 클라이언트 뷰

Bootstrap을 이용하여 반응형 인터페이스를 구현

<br>

###### 안드로이드 클라이언트
![Android-view](/assets/images/easyprint-project/merge.png "android-view"){: width="800" height="2000"}
<br>

###### 데스크탑 클라이언트 
![Desktop-view](/assets/images/easyprint-project/desktop-view.png "desktop-view"){: width="800" height="950"}
<br>
<br>
<br>
<br>




### 어려웠던 점,&nbsp; 어떻게 극복했나
<br>
> ### 문제점 
> **초기 구상 시 서버와 클라이언트와의 통신을 전부 AJAX Long Polling 혹은 Server Side Event 로 해결하려 했으나 
개발을 진행하다 보니 프로그램 구조가 너무 복잡해져 새로운 통신 방식의 필요성을 느낌**   
<br>
> ### 어떻게 해결하였나
> * Websocket으로 통신 방법을 변경했고 Websocket을 지원하는 비동기 서버가 필요해 기존에 사용하던 uwsgi 서버를 eventlet 서버로 변경
> * eventlet 서버를 ssl과 같이 사용하지 못하는 issue가 존재해 nginx를 ssl프록시 서버로 추가사용 

<br>	

> ### 문제점
>  **별도에 로그인 페이지를 두지 않고 메인페이지 왼쪽상단의 사이드바에서 로그인하는 구조이기 때문에 하나의 url에 로그인 폼과 다른 폼이 겹치는 경우가 발생함**       
<br>
> ### 어떻게 해결하였나
> **폼이 겹칠 때에 다음의 코드와 같이 로그인 폼을 데코레이터로 넘겨 처리**                 
<br>
> ###### _로그인 폼 데코레이터_
> 
> ```python
> 
> # Put this decorator on every page which needs login form on their left sidebar
> # Route functions also need to add **kwargs argument
> def add_login_form(original_function):
>     @wraps(original_function)
>     def wrapper(*args, **kwargs):
>         # If user not logged in yet pass the login form through the kwargs argument
>         if current_user.is_anonymous:
>             form = LoginForm() 
>             # When the POST message is sended to the server
>             if form.validate_on_submit():
>                 '''
>                 Do some validation works in here
>                 '''
>             # Pass form as 'form' key
>             kwargs['form'] = form
> 
>         # If user currently logged in then login form is not needed
>         elif current_user.is_authenticated:
>             kwargs['form'] = None
> 
>         return original_function(*args, **kwargs)
> 
>     return wrapper 
> ```
> 
<br>
> **그리고 다음과 같이 blueprint route 함수에서 로그인 폼을 템플릿 렌더링 시에 같이 넘겨준다.**     
> 
> ###### _회원 등록 route_	
> ```python 
> 
> @auth.route('/register', methods=['GET', 'POST'])
> @add_login_form        # Here's the decorator
> def register(*args, **kwargs):    # Pass the kwargs
>     from ..tasks import task_lock
>     reg_form = RegistrationForm()
>     # When the register form is submmited
>     if reg_form.validate_on_submit():
>         ''' 
>         Do some validation works in here
>         '''
>     # Passing login form through register jinja2 template
>     login_form = kwargs['form'] if kwargs['form'] else None 
>     return render_template('auth/register.html', reg_form=reg_form,
>                            login_form=login_form)
> 
> 
> ```
> 
> ~~이러한 디자인 패턴은 나름 기발하다고 생각했었으나 알고보니 flask document에 기술되어 있었다..~~
> 

<br> 
<br> 
<br> 

### 프로젝트의 한계, &nbsp;개선방안

<br>
> ##### 한계점 
>
> 1. 프린팅 도중 서버의 부하가 심해지면 프린팅 퀄리티가 급격히 저하됨.   
> 2. 프린팅이 완료되었을 때  연속적으로 다음 프린팅을 하기 위해서는 출력이 완료된 출력물을 자동을 프린팅 베드에서 떨어뜨리는 일을 해주는 특별한 하드웨어 혹은 [gcode]({{ page.gcode }})가 필요함.    
> 3. 프린팅 도중 웹소켓 connection이 refresh 되면 Malin에서 시리얼로 커맨드를 읽지 못해 프린터가 정지하는 현상이 발생함.   
<br>
<br>
<br>
>
> ##### 개성방안 
> 
> 1. 위에서 언급했다시피 easyprint는 모든 작업을 라즈베리 파이 위의 docker container 위에서 실행함.   
> 애초에 라즈베리파이의 성능이 생각보다 좋지 못함.  특히나 슬라이싱을 해야 할 때에 굉장히 많은 CPU 자원을 소모함.   
> 만약 프린팅 도중에 슬라이싱 이벤트가 많이 발생하면 서버에 부하가 심해져 범용os 특성상 프린팅을 담당하는 프로세스의 스케쥴링 시간을 보장해주지 못함.          
> (linux의 nice 값과 ionice 값을 조절해 프로세스 우선순위를 다르게주면 되지 않을까 했지만 소용없었음)    
> 왜 그런가 하니 3D 프린터의 [Marlin]({{ page.Marlin }}) 펌웨어와 라즈베리파이가 시리얼로 통신을 할 때에 서버의 부하가 심해지면 펌웨어에서 제때 시리얼로 넘어온 값을 읽지 못해 체크섬이 깨지는 오류가 발생   
> 체크섬을 다시 계산하고 다시 보내는 과정에서의 딜레이가 생기고 이런 딜레이가 쌓이게 되면 프린팅이 가끔씩 멈췄다가 동작함    
> 이는 프린팅 퀄리티를 급격히 떨어뜨리는 결과를 가져옴  
> 검색해보니 [octoprint]({{ page.octoprint }})에서도 이러한 경우가 발생하고 [octoprint]({{ page.octoprint }})에서는 프린팅 도중 [octoprint]({{ page.octoprint }}) 프로세스 외의 다른 작업을 하지 않기를 권장함 따라서 이는 리얼타임os가 아니기 때문에 발생하는 설계상의 문제임  
> 결과적으로 이러한 문제를 해결하기 위해서는 프린팅을 담당하는 프로세스와 그 밖의 서비스를 담당하는 프로세스를 물리적으로 분리시키는것이 필요함   
> 프린팅만을 라즈베리파이 위에서 해결하고 나머지는 클라우드 서버 등을 이용하는 것이 좋은 방법일듯함    
<br>
<br>
>
> 2. 다음과 같은 해결 방안이있음 
<br>
> ###### Gcode로 해결
> [![Video Label](http://img.youtube.com/vi/avlengYsJdw/0.jpg)](https://www.youtube.com/watch?v=avlengYsJdw)
> ###### 특별한 하드웨어를 추가하여 해결(컨베이어 벨트형식의 빌드 플레이트를 이용)
> [![Video Label](http://img.youtube.com/vi/-25e5qcELvw/0.jpg)](https://www.youtube.com/watch?v=-25e5qcELvw)
<br>
<br>
>
> 3. 내부적으로 라즈베리파이에서 시리얼 통신과 웹소켓을 동시에 사용할 때 동일한 하드웨어 자원을 참조하는 것이 아닌가하는 유추.     
> 1번의 해결방안과 마찬가지로 궁극적으로는 프린팅 프로세스와 나머지 서비스를 물리적으로 분리시켜야함     

<br>
<br>
<br>
<br>


### 기타 느낀 점
협업을 하는 상황이 아니고 개인이 진행하는 프로젝트라 테스트코드를 작성하는 것에 의미를 두지 않았었음   
테스트코드의 길이가 실제 소스보다 길어질 수 있고 테스트코드를 통과했다고 하더라도 모든 오류를 잡지 못할 것이라 판단해서 unittest를 하지 않았었음   
돌이켜보니 매번 오류를 검증하느라 더 많은 시간이 소비되었음 장기적으로 보았을 때 테스트 코드를 작성하는 것은 시간 낭비가 아님을 깨닫게 됨.     
__TDD는 선택이 아닌 필수였음__           

<br>
그 밖에도 하드웨어를 다룰 때의 범용os의 한계와 리얼타임os의 필요성을 느낌  

<br>
