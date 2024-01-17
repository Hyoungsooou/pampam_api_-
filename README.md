# 팜팜 api 분석
> 개인적인 의견으로써 10일간 팜팜 api 코드를 보며 느낀점입니다.
> <br>크게 분류해서 작성했습니다.
> 

## 개발하는 사람마다 성향이 다릅니다.<br>이게 맞고 저게 맞고하는 잘잘못을 따지는게 아닌 팜팜 코드를 보고 개인적인 느낀점과 개인적인 분석을 한 글임을 알립니다.

- ### 지속적으로 수정하겠습니다.

## 1. Controller

팜팜의 Controller부분 즉 비지니스 로직을 담당하는 부분은 데이터 모델 ( 데이터베이스 테이블)로 구분하여 작성되어있습니다.

#### 느낀점
1. 전반적으로 코드의 흐름을 따라가기가 매우 어렵습니다.
	- 많은 부분이 캡슐화 되어있습니다.
		1. 특히 orm을 담당하는 QueryMaker의 존재이유를 잘 모르겠습니다.
		2. Error 메세지를 만들어주는 클래스 ErrorBuilder의 사용으로 인해 에러 메세지를 읽기가 많이 어려웠습니다.

2. 전반적으로 코드가 구현 이후로 관리가 안된 느낌이 강합니다.
	- 정작 캡슐화가 필요한 부분은 되어있지 않습니다.
```python
def send(self, phone: str, db: Session):
        today = datetime.date(datetime.now())
        phone_check_count = db.query(Verification).filter(
            getattr(Verification, "created_time") >= today,
            getattr(Verification, "phone_number") == phone
        ).count()
        phone_check_count += 1
        passcode = create_passcode(length=6)
        message = ""
        if phone_check_count > 5:
            logging.info(F"[FAIL TO SEND SMS]: phone: {phone} SEND COUNT EXCEED")
            passcode = ""
            message = "일일 문자 인증 건수를 모두 소진하였습니다."
        elif phone_check_count == 5:
            phone_in = {
                'passcode': passcode,
                'phone_number': phone,
                'verify_type': VerifyType.PHONE
            }
            message = "일일 문자 인증 건수를 모두 소진하였습니다."
            send_sms(phone_number=phone, passcode=passcode)
            crud.verification.create(db=db, obj_in=phone_in)
        elif phone_check_count == 4:
            phone_in = {
                'passcode': passcode,
                'phone_number': phone,
                'verify_type': VerifyType.PHONE
            }
            message = "일일 문자인증 가능 건수가 1회 남았어요."
            send_sms(phone_number=phone, passcode=passcode)
            crud.verification.create(db=db, obj_in=phone_in)
        else:
            phone_in = {
                'passcode': passcode,
                'phone_number': phone,
                'verify_type': VerifyType.PHONE
            }
            crud.verification.create(db=db, obj_in=phone_in)
            send_sms(phone_number=phone, passcode=passcode)
        return {"count": phone_check_count, "message": message}
```
- 위와 같은 로직으로 되어있는 부분이 몇몇 보입니다.
	- 개선점
		1. 5개 이상인지 체크하는 분기 다음 분기들에서 phone_in 데이터가 모두 동일합니다.
		2. 분기가 있어야하는 이유는 message의 출력이 달라서인데 이러한 부분을 함수로 묶어서 사용했으면 어땠을까 하는 느낌이 듭니다.

3. Errobuilder의 사용이유
	- 에러 메세지와 에러 코드를 작성하는 부분을 클래스화 시켰습니다.
		1. api를 테스트하면서 에러코드를 확인했을때 처음에 어떤 에러인지 확인하기가 매우 어렵습니다.
```json
{
  "detail": {
    "code": 9192,
    "message": "이메일 또는 패스워드가 올바르지 않습니다. 다시 입력해주세요."
  }
}
```
- 위의 에러 코드와 메세지는 사용자가 로그인을 실패했을때 나타나는 에러메세지입니다.
	- 에러코드 9192번은 코드상에서 존재하지않습니다.
		1. 이러한 문제가 나타나는 이유는 여러가지 에러코드를 표현하기 위해 개발자가 정의한 에러코드들을 모두 더해서 표현하기 때문입니다.
		2. 사용자 정의 에러코드의 복잡성은 올라갔으나 거의 모든 에러 상태 코드를 400으로 리턴합니다.
	- 모든 상태의 에러를 표현하기위해 에러코드를 이렇게 다분화시킨 것의 이유를 모르겠습니다.
		1. 로그를 처리하는 부분이 모두 info()로 처리되어 있습니다.
		2. 에러코드를 다분화 시켜서 얻을 수 있는 이득 (로그 분석의 용이, 프론트와 합의를 통한 메세지 확립)등이 보여지지 않습니다.
			- 모든 팜팜 앱 코드를 확인한 것은 아니지만 실제로 api를 호출할 때 사용자에게 보여지는 에러 메시지를 프론트에서 따로 다시 작업합니다.

```json
{
  "detail": {
    "code": 11021,
    "message": "삭제된 댓글입니다."
  }
}
```
- 위와 같은 에러 메세지를 처음 봤을 때 에러코드 11021이 어떤 상태를 표현하는 건지 명확하지 않았습니다.
- 에러코드의 다분화를 한 이유가 궁금합니다.


- API 엔드포인트만으로 어떨 때 호출해야 할지 명확하지 않은게 보입니다.
```text
POST /v1/comments/bulk
```
- 위 API는 행위 (POST)와 엔드포인트 (/comments/bulk)로 보아 일정 댓글을 한번에 db에 저장하는 엔드포인트 인줄 알았습니다.
	- 이 API는 일정 댓글을 한번에 지워주는 엔드포인트입니다.


## CRUD

데이터를 핸들링하는 메서드를 담당하는 부분입니다.

#### 느낀점
- QueryMaker의 존재 이유를 모르겠습니다.
	- orm을 사용하는 이유 중 하나는 협업에 용이하다는 점입니다.
		1. 많은 부분 filter, get 등을 모두 캡슐화 시켜서 사용하기 떄문에 QueryMaker 메소드를 모른다면 읽기가 매우 불편합니다.
		2. 이 코드를 보는 사람들은 orm뿐만 아니라 QueryMaker도 알아야하는 부담이 생깁니다.
```python
# QueryMaker
deals = crud.deal.get_query_maker(db=db) \
            .add_contains(column="seller_id", value=[current_user.id, block_user_id]) \
            .add_contains(column="buyer_id", value=[current_user.id, block_user_id]) \
            .add_not_contains("status", [Status.OPENED, Status.MESSAGE])\
            .get_all()

# ORM
deals = db.query(Deals)\
		.filter(Deals.seller_id.in_([current_user.id, block_user_id]),
			Deals.buyer_id.in_([current_user.id, block_user_id]),
			~Deals.status.in_([Status.OPENED, Status.MESSAGE]))\
		.all()
```

- 위 코드 예시는 QueryMaker를 ORM으로 변환하여 작성했습니다.
	- 개인적으로 orm만으로 이용하는 것이 협업에 더 용이하다고 생각합니다.

- 무엇을 의도하고 QueryMaker를 작성한 것인지 궁금합니다.


	
## 전체적인 부분
- 너무 모든 기능을 직접 구현하는 경향이 많이 보였습니다.
	- 라이브러리를 가져다 쓰면 쉽게 해결할 수 있는 부분이 많이 보여집니다.
	- 라이브러리를 믿지못(?)하여 직접 구현한 것인지 궁금합니다.
	- 이후 개발할 때도 상용 오픈소스를 사용하면 안되는 것인지도 궁금합니다.


## 총평

전반적으로 정리가 안된 부분들이 많이 보여 코드를 읽고 분석하는데 시간을 많이 할애했습니다.
<br>
이후 이전에 개발했던 방식으로 새로운 API를 개발하려 할때 어려움이 많이 예상이 됩니다.
<br>
인증 서버를 따로 빼서 MSA환경을 구성하려 한 이유도 위의 영향이 큽니다.
<br>
새롭게 개발되는 부분들도 크게 어떤 서비스단위로 개발이 진행될텐데 이때 새롭게 다시 개발하고 싶습니다.

## 인증서버를 분리하면서 얻을 수 있는 이득

1. 생산성이 올라갑니다.
	- 같은 팜팜 도메인을 활용한 새로운 서비스의 개발에서의 생산성이 올라갑니다.
		1. 기존 팜팜 서버와 동일한 위치에서 서비스하게 될 경우 기존 코드를 활용가능한 수준만큼 익히는데 걸리는시간이 많이 듭니다.	
		2. 예상치 못한 충돌(이전 코드와의 호환성)을 예상하고 기회비용을 생각했을 때 새롭게 개발하는게 합리적이라는 생각이 듭니다.

2. 경매 시스템이 가장 많은 트래픽을 감당하게 될텐데(예상대로 진행된다면) 경매 시스템의 트래픽으로 인한 부하가 팜팜 앱에도 전이 될 수 있습니다.

3. 다른 서비스의 확장도 쉽게 가능합니다.
	- 사용자 인증 서버를 분리함으로써 다른 서비스를 개발하는데 오롯이 집중할 수 있습니다.





 
