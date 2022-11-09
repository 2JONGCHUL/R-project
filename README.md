# 602277124 이종철
# 2022-11-09
## [6단계: 클리핑]
```
bnd <- st_read("./01_code/sigun_bnd/seoul.shp")    # 서울시 경계선 불러오기
raster_high <- crop(raster_high, extent(bnd))      # 외곽선 자르기
crs(raster_high) <- sp::CRS("+proj=longlat +datum=WGS84 +no_defs +ellps=WGS84 + towgs84=0,0,0") # 좌표계 정의
plot(raster_high)  # 지도확인
plot(bnd, col=NA, border = "red", add=TRUE)
```
## [7단계: 지도 위에 래스터 이미지 올리기]
```
library(rgdal)    # install.packages("rgdal")
library(leaflet)  # install.packages("leaflet")
leaflet() %>% 
  #---# 베이스맵 불러오기
  addProviderTiles(providers$CartoDB.Positron) %>% 
  #---# 서울시 경계선 불러오기
  addPolygons(data = bnd, weight = 3, color= "red", fill = NA) %>% 
  #---# 레스터 이미지 불러오기
  addRasterImage(raster_high, 
   colors = colorNumeric(c("blue", "green","yellow","red"), 
   values(raster_high), na.color = "transparent"), opacity = 0.4) 
```
## [8단계: 저장하기]
```
dir.create("07_map")  # 새로운 폴더 생성
save(raster_high, file="./07_map/07_kde_high.rdata") # 저장
rm(list = ls()) # 메모리 정리  
```
## [1단계: 데이터 준비]
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) # 작업폴더 설정
load("./06_geodataframe/06_apt_price.rdata")     # 실거래 불러오기
grid <- st_read("./01_code/sigun_grid/seoul.shp")  # 서울시 1km 그리드 불러오기
apt_price <-st_join(apt_price, grid, join = st_intersects)  # 실거래 + 그리드 공간결합
head(apt_price, 2)
```
## [2단계: 마커 클러스터링 옵션 설정]
```
이상치 설정(하위 10%, 상위 90% 지점)
pcnt_10 <- as.numeric(quantile(apt_price$py, probs = seq(.1, .9, by = .1))[1])
pcnt_90 <- as.numeric(quantile(apt_price$py, probs = seq(.1, .9, by = .1))[9])
마커 클러스터링 함수 등록
load("./circle_marker/circle_marker.rdata")
마커 클러스터링 컬러 설정: 상, 중, 하
circle.colors <- sample(x=c("red","green","blue"),size=1000, replace=TRUE)
```
## [3단계: 마커 클러스터링 시각화]  
```
library(purrr)  # install.packages("purrr")
leaflet() %>% 
   오픈스트리트맵 불러오기
  addTiles() %>%  
   서울시 경계선 불러오기
  addPolygons(data = bnd, weight = 3, color= "red", fill = NA) %>%
   최고가 레스터 이미지 불러오기
  addRasterImage(raster_high, 
                 colors = colorNumeric(c("blue","green","yellow","red"), values(raster_high), 
                                       na.color = "transparent"), opacity = 0.4, group = "2021 최고가") %>% 
  급등지 레스터 이미지 불러오기
  addRasterImage(raster_hot, 
                 colors = colorNumeric(c("blue", "green", "yellow","red"), values(raster_hot), 
                                       na.color = "transparent"), opacity = 0.4, group = "2021 급등지") %>%   
   최고가 / 급등지 선택 옵션 추가하기
  addLayersControl(baseGroups = c("2021 최고가", "2021 급등지"), options = layersControlOptions(collapsed = FALSE)) %>%
   마커 클러스터링 불러오기
  addCircleMarkers(data = apt_price, lng =unlist(map(apt_price$geometry,1)), 
                   lat = unlist(map(apt_price$geometry,2)), radius = 10, stroke = FALSE, 
                   fillOpacity = 0.6, fillColor = circle.colors, weight=apt_price$py, 
                   clusterOptions = markerClusterOptions(iconCreateFunction=JS(avg.formula))) 
```
# 2022-11-02
## [1단계: 지역별 평균 가격 구하기]
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) # 작업폴더 설정
load("./06_geodataframe/06_apt_price.rdata")   # 실거래 불러오기
library(sf)    # install.packages("sf") 
grid <- st_read("./01_code/sigun_grid/seoul.shp")     # 서울시 1km 그리드 불러오기
apt_price <-st_join(apt_price, grid, join = st_intersects)  # 실거래 + 그리드 결합
head(apt_price)

kde_high <- aggregate(apt_price$py, by=list(apt_price$ID), mean) # 그리드별 평균가격(평당) 계산
colnames(kde_high) <- c("ID", "avg_price")   # 컬럼명 변경
head(kde_high, 2)     # 확인
```
## [2단계: 그리드 + 평균가격 결합]
```
kde_high <- merge(grid, kde_high,  by="ID")   # ID 기준으로 결합
library(ggplot2) # install.packages("ggplot2")
library(dplyr)   # install.packages("dplyr")
kde_high %>% ggplot(aes(fill = avg_price)) + # 그래프 시각화
  geom_sf() + 
  scale_fill_gradient(low = "white", high = "red")
```
## [3단계: 지도 경계 그리기]
```
library(sp) # install.packages("sp")
kde_high_sp <- as(st_geometry(kde_high), "Spatial")    # sf형 => sp형 변환
x <- coordinates(kde_high_sp)[,1]  # 그리드 x, y 좌표 추출
y <- coordinates(kde_high_sp)[,2] 

l1 <- bbox(kde_high_sp)[1,1] - (bbox(kde_high_sp)[1,1]*0.0001) # 그리드 기준 경계지점 설정
l2 <- bbox(kde_high_sp)[1,2] + (bbox(kde_high_sp)[1,2]*0.0001)
l3 <- bbox(kde_high_sp)[2,1] - (bbox(kde_high_sp)[2,1]*0.0001)
l4 <- bbox(kde_high_sp)[2,2] + (bbox(kde_high_sp)[1,1]*0.0001)

library(spatstat)  # install.packages("spatstat")
win <- owin(xrange=c(l1,l2), yrange=c(l3,l4)) # 지도 경계선 생성
plot(win)         # 지도 경계선 확인
rm(list = c("kde_high_sp", "apt_price", "l1", "l2", "l3", "l4")) # 변수 정리
```

## [4단계: 밀도 그래프 표시]
```
p <- ppp(x, y, window=win)  # 경계창 위에 좌표값 포인트 생성
d <- density.ppp(p, weights=kde_high$avg_price, # 포인트를 커널밀도 함수로 변환
                 sigma = bw.diggle(p), 
                 kernel = 'gaussian')  
plot(d)   # 확인
rm(list = c("x", "y", "win","p")) # 변수 정리
```
# 2022-10-26
## [1단계: 데이터 불러오기]
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
load("./04_preprocess/04_preprocess.rdata")    # 주소 불러오기
load("./05_geocoding/05_juso_geocoding.rdata") # 좌표 불러오기
```
## [2단계: 주소 + 좌표 결합]
```
library(dplyr)   # install.packages('dplyr')
apt_price <- left_join(apt_price, juso_geocoding, 
                       by = c("juso_jibun" = "apt_juso")) # 결합
apt_price <- na.omit(apt_price)   # 결측치 제거
```

# 6-3: 지오 데이터프레임 만들기

## [1단계: 지오데이터프레임 생성]
```
library(sp)    # install.packages('sp')
coordinates(apt_price) <- ~coord_x + coord_y    # 좌표값 할당
proj4string(apt_price) <- "+proj=longlat +datum=WGS84 +no_defs" # 좌표계(CRS) 정의
library(sf)    # install.packages('sf')
apt_price <- st_as_sf(apt_price)     # sp형 => sf형 변환
```
## [2단계: 지오데이터프레임 시각화]
```
plot(apt_price$geometry, axes = T, pch = 1)        # 플롯 그리기 
library(leaflet)   # install.packages('leaflet')   # 지도 그리기
leaflet() %>% 
  addTiles() %>% 
  addCircleMarkers(data=apt_price[1:1000,], label=~apt_nm) # 일부분(1000개)만 그리기
```
## [3단계: 지오 데이터프레임 저장]
```
dir.create("06_geodataframe")   # 새로운 폴더 생성
save(apt_price, file="./06_geodataframe/06_apt_price.rdata") # rdata 저장
write.csv(apt_price, "./06_geodataframe/06_apt_price.csv")   # csv 저장
```

# 2022-10-12
## 필요한 칼럼 추출하기
```
apt_price <- apt_price %>% select(ymd, ym, year, code, addr_1, apt_nm, 
                                  juso_jibun, price, con_year,  area, floor, py, cnt) # 칼럼 추출
head(apt_price, 2)  # 확인
```

## 전처리 데이터 저장하기
```
setwd(dirname(rstudioapi::getSourceEditorContext()$path))
dir.create("./04_preprocess")   # 새로운 폴더 생성
save(apt_price, file = "./04_preprocess/04_preprocess.rdata") # 저장
write.csv(apt_price, "./04_preprocess/04_preprocess.csv") 
```

## 지오코딩 준비
```
1. 카카오 개발자 사이트(developers.kakao.com) 에서 카카오 계정으로 로그인
2. 내 애플리케이션 클릭
3. 애플리케이션 추가하기 앱이름, 사업자명 입력후 저장
4. REST API 키 발급
```

## 예외처리
```
tryCatch({}) 로 예외처리하여 오류가 발생하여도 반복문을 멈추지 않고 다음 반복으로 건너뛰도록함
```

# 2022-10-05
## 데이터 100건으로 수정
```
datelist <- seq(from = as.Date('2021-01-01'), # 시작
                to   = as.Date('2021-04-30'), # 종료 100개
                by    = '1 month')
```
## 통합데이터 저장하기
```
dir.create("./03_integrated")   # 새로운 폴더 생성
save(apt_price, file = "./03_integrated/03_apt_price.rdata") # 저장
write.csv(apt_price, "./03_integrated/03_apt_price.csv") 
```

## 수집한 데이터 불러오기
```
options(warn=-1) -경고 메시지 무시
table(is.na(apt_price)) - 결측값 False, True 로 확인가능 - True 가 결측값(NA) 갯수
apt_price <- na.omit(apt_price) - 결측값 제거
table(is.na(apt_price) - 결측값 확인
```
## 문자열 공백 처리
```
library(stringr) -> 문자열 처리 패키지
apt_price <- as.data.frame(apply(apt_price, 2, str_trim)) -> 공백제거( 1이면 행, 2면 열에 적용)(str_trim은 함수)
```
## 데이터 변환
```
sub(",","",.) -> 쉼표 제거
as.numeric() -> 문자를 숫자로 변환
```

# 2022-09-28
## LIMITED NUMBER OF SERVICE REQUESTS EXCEEDS ERROR 에러
```
#3단계 : 수집 기간 설정 ( 21-04-30 날짜 수정 100개)
datelist <- seq(from = as.Date('2021-01-01'),
                to = as.Date('2021-04-30'),
              
```
### 2단계: URL 요청 - XML 응답
```
for(i in 1:length(url_list)){   # 요청목록(url_list) 반복
  raw_data[[i]] <- xmlTreeParse(url_list[i], useInternalNodes = TRUE,encoding = "utf-8") # 결과 저장
  root_Node[[i]] <- xmlRoot(raw_data[[i]])	# xmlRoot로 추출
 ```
### 3단계: 전체 거래 건수 확인
```
  items <- root_Node[[i]][[2]][['items']]  # 전체 거래내역(items) 추출
  size <- xmlSize(items)                   # 전체 거래 건수 확인    
```  
 ### 4단계: 거래 내역 추출
 ``` 
  item <- list()  # 전체 거래내역(items) 저장 임시 리스트 생성
  item_temp_dt <- data.table()  # 세부 거래내역(item) 저장 임시 테이블 생성
  Sys.sleep(.1)  # 0.1초 멈춤
  for(m in 1:size){  # 전체 거래건수(size)만큼 반복
  ``` 
  ### 세부 거래내역 분리
  ```
    item_temp <- xmlSApply(items[[m]],xmlValue)
    item_temp_dt <- data.table(year = item_temp[4],     # 거래 년 
                               month = item_temp[7],    # 거래 월
                               day = item_temp[8],      # 거래 일
                               price = item_temp[1],    # 거래금액
                               code = item_temp[12],    # 지역코드
                               dong_nm = item_temp[5],  # 법정동
                               jibun = item_temp[11],   # 지번
                               con_year = item_temp[3], # 건축연도 
                               apt_nm = item_temp[6],   # 아파트 이름   
                               area = item_temp[9],     # 전용면적
                               floor = item_temp[13])   # 층수 
    item[[m]] <- item_temp_dt}    # 분리된 거래내역 순서대로 저장
}
    apt_bind <- rbindlist(item)     # 통합 저장
```
### CSV로 저장
```
 region_nm <- subset(loc, code== str_sub(url_list[i],115, 119))$addr_1 # 지역명 추출
    month <- str_sub(url_list[i],130, 135)   # 연월(YYYYMM) 추출
    path <- as.character(paste0("./02_raw_data/", region_nm, "_", month,".csv")) # 저장위치 설정
    write.csv(apt_bind, path)     # csv 저장
    msg <- paste0("[", i,"/",length(url_list), "] 수집한 데이터를 [", path,"]에 저장 합니다.") # 알림 메시지
    cat(msg, "\n\n")
    }   # 바깥쪽 반복문 종료
```

# 2022-09-21
### 인증키 입력 (공공데이터포털 일반 인증키 Encoding)
```
service_key <- "68XcETqD8F%2FF8bzG7Zie6O7Jf32O85r2E8QSXx09zEfhZ7DW9qxo1GgqA52x3ayOSHKvGrRftnqzfWvNapoceA%3D%3D"  # 인증키 입력
```

### 1단계: 요청목록 만들기
```
url_list <- list() # 빈 리스트 만들기
cnt <-0	           # 반복문의 제어 변수 초깃값 설정
```

### 2단계: 요청목록 채우기
```
for(i in 1:nrow(loc)){           # 외부반복: 25개 자치구
  for(j in 1:length(datelist)){  # 내부반복: 12개월
    cnt <- cnt + 1               # 반복누적 카운팅
    
  ## 요청 목록 채우기 (25 X 12= 300)
    url_list[cnt] <- paste0("http://openapi.molit.go.kr:8081/OpenAPI_ToolInstallPackage/service/rest/RTMSOBJSvc/getRTMSDataSvcAptTrade?",
                            "LAWD_CD=", loc[i,1],         # 지역코드
                            "&DEAL_YMD=", datelist[j],    # 수집월
                            "&numOfRows=", 100,           # 한번에 가져올 최대 자료 수
                            "&serviceKey=", service_key)  # 인증키
  } 
  Sys.sleep(0.1)   # 0.1초간 멈춤
  msg <- paste0("[", i,"/",nrow(loc), "]  ", loc[i,3], " 의 크롤링 목록이 생성됨 => 총 [", cnt,"] 건") # 알림 메시지
  cat(msg, "\n\n") 
}
```
### 3단계: 요청 목록 동작 확인
```
length(url_list)                # 요청목록 갯수 확인
browseURL(paste0(url_list[1]))  # 정상작동 확인(웹브라우저 실행)
```
### 1단계: 임시 저장 리스트 생성
```
# install.packages("XML")
# install.packages("data.table")
# install.packages("stringr")

library(XML)        # install.packages("XML")      
library(data.table) # install.packages("data.table")
library(stringr)    # install.packages("stringr")

raw_data <- list()        # xml 임시 저장소
root_Node <- list()       # 거래내역 추출 임시 저장소
total <- list()           # 거래내역 정리 임시 저장소
dir.create("02_raw_data") # 새로운 폴더 만들기
```

# 2022-09-14

### 1단계: 작업 디렉토리 설정
```
#install.packages("studioapi")
setwd(dirname(rstudioapi::getSourceEditorContext()$path)) #작업폴더 설정
getwd() #작업폴더 확인 
```

### 2단계 : 수집 대상지역 설정
```
loc <- read.csv("./sigun_code.csv", fileEncoding = "utf-8") #지역코드
loc$code <- as.character(loc$code) #행정구역명 문자 별환
head(loc, 2) #확인
```

### 3단계 : 수집 기간 설정
```
datelist <- seq(from = as.Date('2021-01-01'),
                to = as.Date('2021-12-31'),
                by = '1 month')
datelist <- format(datelist, format = '%Y%m')
datelist[1:3]
```

# 2022-09-07

- 공공데이터포털 (https://www.data.go.kr/)
- 국토교통부_아파트매매 실거래자료
- 참고문서 : 아파트 매매 신고정보 조회 기술문서.hwp

일반 인증키 (Encoding) : 68XcETqD8F%2FF8bzG7Zie6O7Jf32O85r2E8QSXx09zEfhZ7DW9qxo1GgqA52x3ayOSHKvGrRftnqzfWvNapoceA%3D%3D

- http://openapi.molit.go.kr:8081/OpenAPI_ToolInstallPackage/service/rest/RTMSOBJSvc/getRTMSDataSvcAptTrade?LAWD_CD=11110&DEAL_YMD=201512&serviceKey=68XcETqD8F%2FF8bzG7Zie6O7Jf32O85r2E8QSXx09zEfhZ7DW9qxo1GgqA52x3ayOSHKvGrRftnqzfWvNapoceA%3D%3D

