# 2022_OSSCA_SPRINT
### 2022년 오픈소스 컨트리뷰션 아카데미 지역스프린트 @춘천

# [ PR했던 코드 분석 및 진행과정 ]
### 1. `array.ArrayType` Error
- Difficulty : very easy

- FileName : `array.rs` 

- Reaseon : RustPython \> AttributeError: module 'array' has no attribute 'Arraytype'  
        &nbsp;&nbsp;array모듈에 ArrayType 기능이 없음
            
- Issue Tracing :
  1. array.rs 파일로 가서
  2. #[pyattr(name = "ArrayType")]를 추가함

- Result : Error Fixed! ArrayType만 추가하면 되는 문제였음!

### 2. `test_builtin::test_format` Error
- Difficulty : easy

- FileName : `test_builtin.py`

- Error Location :  
def test_bytearray_translate(self):  
        x = bytearray(b"abc")  
         self.assertRaises(ValueError, x.translate, b"1", 1) # <- error  
         self.assertRaises(TypeError, x.translate, b"1"*256, 1)  
        
- Reason : CPython과 RustPython에서의 오류를 비교함  
CPython \> ValueError: translation table must be 256 characters long O  
정상 : 정상적으로 ValueError로 처리됨  

  RustPython \> TypeError: 'int' object is not iterable X   
  에러 : ValueError로 처리되어야 하는데 TypeError로 처리됨

- Issue Tracing :
  1. ValueError가 떠야하는데 TypeError가 뜸    [File: test_builtin.py, line 1702]
  2. bytearray의 translate에 문제가 있는 것 같음    [File: test_builtin.py, line 1702]
  3. translate함수 내부의 ByteInnerTranslateOptions 확인    [File: bytearray.rs, line: 500]
  4. bytesinner.rs의 pub struct ByteInnerTranslateOptions확인    [File: bytesinner.rs, line: 203]
  5. ByteInnerTranslateOptions는 table이 올바른 타입이 아니면 자동으로 TypeError반환(러스트 특징) [File: bytesinner.rs, line:205]
  6. TypeError를 자동으로 반환하지 않도록 하고 ValueError로 반환하도록 고쳐야함 
  7. table먼저 수정함
  8. pub struct ByteInnerTranslateOptions { ~ }의 table: Option\<PyBytesInner\>에서 Option\<PyObjectRef\>로 변환,  
     최상위 타입인 PyObjectRef로 먼저 받아서 자동으로 TypeError가 반환되는 것을 막음  [File: bytesinner.rs, line:205]
  9. impl ByteInnerTranslateOptions { ~ }은 table이 PyBytesInner 인것으로 생각하고 구현됬으니 바꿔주어야 함  
     [File: bytesinner.rs, line:214]
  10. v의 상태를 PyObjectRef로 받음    [File: bytesinner.re, line: 214]
  11. 그리고 다시 PyBytesInner로 바꿈  
      let v: PyBytesInner = v.try_into_value(vm).map_err(|_| {vm.new_value_error("translation table must be 256 characters long".to_owned())})?;  [File: bytesinner.rs, line:215]
  12. ByteInnerTranslateOptions의 table에서 ValueError를 반환하도록함    [File: bytesinner.rs, line:215]
  13. delete에서도 똑같이 바꿔줌
  14. delete: OptionalArg\<PyBytesInner\>에서 OptionalArg\<PyObjectRef\>로 바꿔줌    [File: bytesinner.rs, line 207]
  15. table에서와 같이 delete에서도 let byte: PyBytesInner = byte.try_into_value(vm)?; 로 PyObjectRef로 받은걸 다시 PyBytesInner로 바꿔줌 [File: bytesinner.rs, line:227]
  16. 수정 끝
  
- Additional modifications : 
  Reason - 정윤원 멘토님이 중복되는 에러 핸들링 부분들을 통합할 수 있다고 추가 수정사항을 주셨음  
  let bytes = v  
                    .try_into_value::\<PyBytesInner\>(vm)  
                    .ok()  
                    .filter(|v| v.elements.len() == 256)  
                    .ok_or_else(|| {  
                        vm.new_value_error(  
                            "translation table must be 256 characters long".to_owned(),  
                        )  
                    })?;  
                Ok(bytes.elements.to_vec())  
- Result :  
           러스트도 낯설었고 파이썬의 bytearray 기능도 써본 적이 없는 기능이라 많이 고생하면서 고쳤다.  
           문제가 발생하는 위치를 따라가면서 어떤 코드가 문제인지 추적했고 그 과정에서 러스트의 match구문과 대수식 열거형에 대해 알 수 있었다.  
           처음해보는 방식의 코딩이라 어렵기도하고 해매기도 했지만 멘토님들이 많이 도와주셔서 무사히 고칠 수 있었고 Pull Request도 할 수 있었다!!

### 3. `test_long::test_from_bytes` Error 
- Difficulty : medium
 
- FileName : `test_long.py`
  
- Error Location : 
def test_from_bytes(self):
....
self.assertRaises(TypeError, int.from_bytes, "", 'big')    [File: test_long.py, line: 1339]
  
- Reason :  
CPython \> TypeError: cannot convert 'str' object to bytes  
정상 : str객체 때문에 TypeError가 발생해야 함  

  RustPython \> AssertionError: TypeError not raised by from_bytes  
  에러 : from_bytes에서 TypeError가 발생하지 않아서 self.assertRaises가 AssertionError를 반환함  
  
- Issue Tracing : 
  1. TypeError를 출력해야하는데 TypeError가 발생하지 않아서 AssertionError가 발생함    [File: test_long.py, Line: 1339]
  2. int함수 내부의 from_bytes에 문제가 있는 것 같음    [File: int.rs, Line: 626]
  3. from_bytes의 args: IntFromBytesArgs에 문제가 있는 것 같음    [File: int.rs, Line: 628]
  4. struct IntFromByteArgs의 PyBytesInner에 문제가 있는 것 같음    [File: int.rs, Line: 763] 
  5. byteinnere.rs의 bytes_from_object로 넘어옴    [File: bytesinner.rs, Line: 1203]
  6. 잘못된 객체인 경우 Err()로 보내기 위해 코드를 추가함  
     if !obj.fast_isinstance(&vm.ctx.types.str_type) { # <- New Add  
           if let Ok(elements) = vm.map_iterable_object(obj, |x| value_from_object(vm, &x)) {  
               return elements;  
           }  
     }  
     obj가 str_type인 경우는 반환 코드를 실행하지 않음.                                                       
  7. 수정 끝

- Result :  
           보통 난이도라 다 못하지 않을까 걱정했었지만 앞선 문제들을 푼 것처럼 하나씩 따라가며 고쳐갈 수 있었다. 수정하면서 러스트의 map_or() 함수에
           대해 알 수 있었다.      

### 소감
처음엔 솔직히 지원서를 쓰고도 많이 걱정했다. \"볼 것도 없이 탈락하지 않을까\", \"참여한다해도 멘토, 멘티분들에게 피해만 끼치지 않을까\" 하는 불안감이 있었는데 
다행히도 멘토분들께서 전혀 개의치 않고 도와주셨고 덕분에 열정적으로 컨트리뷰션에 참여할 수 있었다.  
러스트라는 언어도 생소했고 파이썬도 그다지 잘하는 편은 아니었기에 처음엔 고칠 오류를 선택하는 것부터 난관이었다. 코드의 어떤 부분이 문제인지, 어떻게 고쳐야 할지가 전혀 생각나지 않았었고 오랜만에 쓰는 VSCode와 Git Bash는 이러한 어려움을 더더욱 늘렸다.   
하지만 멘토분들이 열정적으로 계속 지도해주신 덕분에 2일차부터는 어느정도 혼자서 문제 해결에 대해 추론해 볼 수 있었고 결국 컨트리뷰션 기간 동안 3개의 Pull Request를 달성할 수 있었다!!  
직접 프로젝트에 기여한다는 기쁨에 대해 알 수 있었고 test_bytearray_translate Error PR에 다른 개발자분이 VSCode로 작업할 때 활용할 수 있는 기능에 대해 알려주시기도 했다!  
컨트리뷰션 기간동안 온전히 컨트리뷰션에 집중할 수 있도록 지원해주신 멘토님들과 오픈소스 컨트리뷰션 아카데미 운영진분들 덕분에 정말 뜻깊은 경험이 될 수 있었고  
이번에 멘토로 참가하신 신지홍 멘토님께서 원래 멘티로 참여하셨다가 멘토가 되신것 처럼 나도 언젠가 멘티에서 멘토가 된다면 좋을 것 같다!  
정말 재밌었고 좋은 경험이었습니다! 감사합니다!!!                                                            
      
# [ Pull Request ]
                                                            
# [ 사진 ]                                                             
