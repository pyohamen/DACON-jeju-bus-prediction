# 퇴근시간 버스승차인원 예측

> 쥬피터 파일의 내부에서 read_csv() 함수의 모든 csv 파일들은 아래 링크에서 받으실 수 있습니다. 

+ 데이콘 데이터 다운로드
  - 퇴근시간 버스승차인원 예측 경진대회: https://dacon.io/competitions/official/229255/data/
  - KCB 금융스타일 시각화 경진대회: https://dacon.io/competitions/official/82407/data/
+ 외부 데이터
  - 기상 데이터: [weather.csv](weather.csv), [rain.csv](rain.csv)
  - 위치 관련 데이터: [df_location.csv](df_location.csv), [life_location.csv](life_location.csv)



# 1. Aim

퇴근시간의 버스 승차인원을 예측하자

지도학습이니 회귀를 이용해 예측을 이뤄내보자.



# 2. 평가척도

RMSE

> MSE (Mean square error) 를 루트 씌어준 것

![](https://firebasestorage.googleapis.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-MKOtwx4yI7B68O1bBrp%2Fsync%2F5b7fb9629ba7d68ed5a6cb6c27798144c1413153.png?generation=1617087704636147&alt=media)

이를 최소화하도록 접근해보자.



# 3. EDA

> Exploratory Data Analysis
>
> 크게 요약 (Summary) 와 시각화 (Visulize) 로 나뉜다.



## 3.1 EDA (Summary)

**train/test**

- train 에는 출근시간은 승차, 하차 모두 있지만, 퇴근시간은 승차 인원만 있다, (결측치 없음!)
- test 에는 출근시간의 승차, 하차 만 있다. (마찬가지로 결측치 없음)

**bts**

- 하차할 때 카드를 찍지 않은 것에 대한 결측치 존재
- bts 는 train/test 데이터의 승차 정보를 갖고 있다.

**jeju_life**

- 관련있는 변수만으로 전처리 해서 병합하자

**weather**

- 오전 10시의 기상정보만 갖고있다.
- 날짜변수를 기준으로 train/test 에 병합하자

**rain**

- 지점 변수에 대해 test data 지점별 기상정보에 추가하자



## 3.2 EDA (Visualize)

- 대부분의 퇴근시간 승차인원은 0~5 명 구간
- 수요일까지 점차 증가하다가 감소
- 시외가 약 2배 많음
- 출근의 탑승객과 거의 유사



# 4. Making feature & Model

> 학습에 의미가 있을법한 피쳐들을 만들고, 모델을 만들어보자



**train / test 구분 피쳐 생성**

> 학습데이터로 만든 모델에 테스트 데이터를 입력으로 넣을 것이기 때문에, 두 데이터는 변수들이 같다. 따라서 구분할 수 있는 cue 변수를 추가

```python
train['cue'] = 0
test['cue'] = 1

# 학습 데이터와 테스트 데이터 통합
df = pd.concat([train, test], axis=0)
```



## 4.1 내부 데이터를 통해 변수 생성



### 4.1.1 EDA 를 통한 변수

1. **요일 변수 ( weekday, weekdaymean )**
   - `pd.to_datetime()` datetime 변수형으로 변환
   - `datetime.weekday` : 요일 추출
   - 각 요일에 해당하는 인덱스들을 추출하여 요일별 평균 탑승 승객수값을 가지는 **weekdaymean** 피쳐를 생성



2. **버스 종류별 평균 탑승객 수 (in_out_mean)**
   - 위처럼 버스 종류별로 인덱스 추출하여 평균 값 **in_out_mean** 피쳐를 생성



3. **일별 오전 시간대의 총 탑승객 수 (date, x~y_ride_sum)**
   - 날짜별로 총합을 구해 일별 오전 시간대별 총 탑승객 수를 갖는 피쳐들을 생성



### 4.1.2 도메인 조사를 통한 변수 생성

1. **배차간격 (bus_interval)**
   - 수요에 따라 배차 간격이 짧기 때문에 위 변수를 생성



2. **수요가 많을 것으로 예상되는 정류장 (school, transfer)**
   - 정류장명에 '학교' 가 포함되어 있는지 아닌지에 대한 피쳐 생성



3. **연휴 (holiday)**
   - 공휴일인지 아닌지에 대해서 구분하는 피쳐 생성



### 4.1.3 시간대를 활용한 변수 생성

> 여기서 데이터 누수가 발생하지 않도록, 오전시간대만의 데이터만 활용해야 한다.



1.  **t~t+2 승하차 시간대 통합 변수(68a, 810a, 1012a, 68b, 810b, 1012b)**

   > 승차 시간간격(1h)과, 예측해야 하는 하차 시간간격(2h) 이 다르기 때문에 이를 통합하자.



2. **오전 시간의 승객 수(ride_sum, takeoff_sum)**

   > EDA - Visualize 에서 확인한대로 오전 시간의 승하차 인원 (출근) 인원이 많다면, 퇴근 시간의 승차 인원 (퇴근) 인원도 클 경향을 보일 것입니다.



### 4.1.4 bus_bts 를 활용한 변수

> EDA - Summary 에서 보았듯이 bus_bts.csv 를 통해 탑승한 승객의 정보를 추가로 파악할 수 있다. 승객 정보별 승객 수의 합과 비율 변수를 만듭니다.



1. **카테고리별 승객 수의 합과 비율 (adult, kids, teen, elder, adult_prop, kids_prop, teen_prop, elder_prop)**

   > 어느 정류장에 어떤 승객이 탑승하는지 알 수 있음



### 4.1.5 좌표를 활용한 변수

> 버스 정류장의 위치는 버스 승차 인원에 영향을 준다. 버스 정류장의 좌표를 활용해 위치 정보를 추가하는 변수 생성



1. **버스 정류장과 인구 밀집 지역 사이 거리 (dis_jejusi, dis_seoquiposi)**

   > 제주시와 서귀포시 는 인구가 많으니 서귀포시와 제주시까지의 거리 변수 2개 생성

2. **탑승하는 승객의 수가 많은 버스 정류장과의 거리**

   > 승차 인원 상위 10개의 버스 정류장 좌표로부터의 거리를 변수 10개 생성



## 4.2외부 데이터를 통한 변수 생성

- 법적 제약이 없는지
- 데이터 누수에 해당하지 않는지
- 기존 df 와 병합하는 과정 주의
  - 정보 손실 최소화하도록 기준 변수 선택
  - 병합하는 과정에서 발생하는 결측치 처리

### 4.2.1 날씨를 활용한 변수

> 4 곳의 기상관측소가 있으니, 우선적으로 각 버스정류장별 가장 가까운 측정소에 대한 변수를 먼저 생성

1. **버스 정류장과 각 측정소의 거리 (dis_jeju, dis_gosan, dis_seongsan, dis_po)**

2. **버스 정류장과 가장 가까운 측정소 (dist_name)**

   > 위 거리 변수를 활용해 가장 가까운 측정소를 알아내자

3. **지점별 기상정보 변수 (temperature, rainfall)**

   > 누수를 방지하기 위해 오전 6 ~ 11 에 해당하는 정보만 편집

4. **비의 유무를 나타내는 변수 (rainy_day)**

   > 기상조건이 여러개 있지만, 그 중 비가 오고 안 오고가 critical 하기 때문에 따로 만듦



### 4.2.2 financial_life_data 를 활용한 변수

> 외부 데이터를 병합하려면 병합 기준이 되는 공통 변수가 하나 이상 있어야 함

> 위도, 경도 변수
>
> ​	기존의 df 의 위도 경도 변수와 jeju_financial_life_data 의 위도, 경도 변수의 형태가 달라 공통된 키가 될 수 없습니다.
>
> 행정구역 이름 변수
>
> ​	두 데이터 모두 행정 구역 이름 변수가 없지만, 위도/경도 변수를 통해 행정 구역 이름을 추출할 수 있습니다. 이 방법은 `3_jeju_life_location.ipynb` 파일에서 각 데이터의 위도/경도 변수를 통해 `df_location.csv` 와 `life_location.csv.csv` 파일을 생성하고 난 후, 수행



**동, 시 별로 직업군, 평균 소비 등등의 경제지표들의 피쳐를 생성**



## 4.3 추가 변수 생성

1.  **승하차 시간대 통합 변수 (t ~ t+3)**

2. **제주도 기상정보 변수 (['date','description','temp','feels_like','rain'])**

3. **id 별 퇴근 시간 총/평균 승객수**

4. **라벨 인코딩**

   **['description', 'bus_route_id','station_code']** 에 대해서 라벨 인코딩





### 4.3.1 라벨 인코딩과 원핫 인코딩 변수

> 머신러닝에서 변수를 활용하려면 변수는 숫자형이어야 한다. 고로 value 가 문자형인 변수들을 라벨 인코딩과 원핫 인코딩을 통해 숫자형으로 변환

**4.3.1.1 라벨 인코딩 변수**

> 문자형 value 들을 등장하는 순서대로 넘버링 해서 숫자형으로 변환

1. **시내 시외 (in_out)**
2. **주중 및 주말 (weekend)**
3. **시, 동 (si, dong)**

##### 4.3.1.2 원핫 인코딩 변수

> 특징이 존재하면 1, 특징이 존재하지 않으면 0 / 이런 식으로 범주형 변수를 수치형으로 변환한다. 따라서 범주의 범위 만큼 더미 변수가 생긴다. 범주의 범위 중 하나의 피쳐에 대해서 1이 되기 때문, 아래 head() 를 보면 이해가 쉽다



# 5. 모델 구축과 검증

> 문자 형태의 변수 제거, datetime 변수 제거

행,  열 확인

```python
display(X_train.shape)
display(X_test.shape)
display(y_train.shape)
```

## 5.1 머신러닝 모델

> 머신러닝의 모델을 크게 나누어 단일 모델과 앙상블 모델이 있는데, 앙상블은 여러 개의 모델을 적절히 결합해 성능 향상을 꾀한 것이다. 앙상블에는 배깅과 부스팅 방법이 있다.

### 5.1.1 배깅 방식 앙상블 모델

> 매 번 조금씩 변수에서 가져오는 값을 다르게 해서 몇 개의 모델을 만든다. 각 모델이 설명변수가 다르기 때문에 당연히 각 모델의 예측값이 조금씩 다르게 된다. 학습의 종류가 분류일 때는 다수결로 예측값을 결정하며, 회귀일 때는 평균값으로 예측값을 결정한다

1. **랜덤 포레스트 (Random Forest)**

### 5.1.2 부스팅 방식 앙상블 모델

> 순차적으로 모델을 학습하면서, 매 번 예측 데이터가 오답인 데이터에 가중치를 주고 그 상태에서 다시 모델을 학습해나가는 것이다.

1. **XGBoost**

   > 부스팅과정에서 순차적으로 다음 모델을 만들 때 negative gradient 를 이용해 만든다. loss function 을 정의하고 이에 대한 negative gradient 가 잔차 (residual) 이 되는 loss function 이 있다. 이 방식을 사용한 것을 그레디언트 부스트 (Gradient Boost) 모델이라고 하는데, 이 모델을 병렬적으로 학습할 수 있게 한 모델이 XGBoost 이다

2. **LightGBM**

   > XGBoost 에 비해 학습 시간을 개선시킨 모델 XGBoost 는 대칭적으로 트리가 성장하지만, LightGBM 은 비대칭적으로 트리가 성장함

## 5.2 모델 검증

여기서 모델 검증은 굉장히 중요하다. 아래 개념들을 잘 숙지하길 바란다.

- 자 여러 모델을 만들었어. 하지만, 어떤 척도로 이 모델의 성능을 검증하지?
- 예측값과 비교해볼 실제 값이 없기 때문에 이를 알 수 없다. 그렇다면 갖고 있던 train data 와 비교하면 어떨까?
- 언뜻 맞는 말 같지만, 학습 데이터에 맞추어 튜닝하고 성능을 높일 시, 과적합 (Overfitting) 이 발생한다.

### 5.2.1 K-fold 교차검증

> 이를 보완하기 위한 K-fold 교차검증 데이터를 여러개로 나누어 그 중 하나씩 test data 로 활용해 성능을 평가하고 이 작업을 K 개로 나누어진 데이터 하나하나에 대해 실행한 후 평균을 내는 것이다.



## 5.3 변수 선택

> 변수를 추가하고 삭제해가면서 성능에 따라 적용한다. 각 모델에 따라 변수 선택을 따로 해준다.

## 5.4 하이퍼파라미터 튜닝

> 트리는 최대 깊이를 얼마나 할 것인지, 불순도향상률 한도를 어떻게 할 것인지에 따라 다양한 트리가 결정된다. 이를 Greedy Search 함으로써 가장 높은 예측력을 갖는 의사결정 나무의 옵션을 찾아내자

## 5.5 최종 모델 구축

### 5.5.1 주 모델 선택

> 랜덤포레스트를 주 모델로 선택

- 근거
  - 부스팅 방법이 오답에 가중치를 줘가면서 모델을 형성하는데, 주어진 데이터의 학습 기간이 짧기 때문에 (9월 한 달) 과적합의 위험이 있음.
  - 랜덤포레스트는 변수 중요도를 볼 수 있음



# 6 성능 향상

## 6.1 앙상블

> 분류 문제 같은 경우는 모델의 결과값들의 분산이 크기 때문에, 여러 모델에 대하여 voting 또는 avg 를 취하는 앙상블 기법을 통해 성능 향상을 꾀할 수 있다.



# 7. 결과

> Public 에서 3등

![](../img/board.png)

