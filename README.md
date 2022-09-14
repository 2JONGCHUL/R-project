# R-project

# 2022-09-14
```
## 1단계: 작업 디렉토리 설정
#install.packages("studioapi")
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) #작업폴더 설정
getwd() #작업폴더 확인 
```


## 2단계 : 수집 대상지역 설정
```
loc <- read.csv("./sigun_code.csv", fileEncoding = "utf-8") #지역코드
loc$code <- as.character(loc$code) #행정구역명 문자 별환
head(loc, 2) #확인
```

# 2022-09-07

- 공공데이터포털 (https://www.data.go.kr/)
- 국토교통부_아파트매매 실거래자료
- 참고문서 : 아파트 매매 신고정보 조회 기술문서.hwp

일반 인증키 (Encoding) : 68XcETqD8F%2FF8bzG7Zie6O7Jf32O85r2E8QSXx09zEfhZ7DW9qxo1GgqA52x3ayOSHKvGrRftnqzfWvNapoceA%3D%3D

- http://openapi.molit.go.kr:8081/OpenAPI_ToolInstallPackage/service/rest/RTMSOBJSvc/getRTMSDataSvcAptTrade?LAWD_CD=11110&DEAL_YMD=201512&serviceKey=68XcETqD8F%2FF8bzG7Zie6O7Jf32O85r2E8QSXx09zEfhZ7DW9qxo1GgqA52x3ayOSHKvGrRftnqzfWvNapoceA%3D%3D

