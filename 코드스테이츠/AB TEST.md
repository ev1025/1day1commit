# 파이썬공부 10일차(2022.08.12)
### AB테스트         
A(Ho)와 B(Ha)를 비교하여 Ha가 맞는지 검증하는 테스트  

---
요약정리
1. 가지고있는 데이터(샘플)의 전환률 차이ctr_diff = (실험군 전환률 - 대조군 전환률)을 구한다. # 샘플의 전환률차이
2. 샘플을 부트스트래핑 하여 모집단을 추정하고, 모집단의 전환률 차이p_diff = (실험군 전환률 - 대조군 전환률)를 구한다 # 모집단의 전환률 차이
3. (모집단전환률>샘플전환률).mean() = p-value (샘플이 모집단을 대표할 수 있는지 p-value를 통해 확인)
4. p-value가 0.05 이하이면 샘플이 모집단의 대표성을 가지게 되고, 샘플데이터의 p-value에 따라 귀무가설을 채택할 지, 대립가설을 채택할 지 정함
------
진행 전 확인사항
1. 샘플 사이즈(방문자 수) : 충분한 데이터가 있어야 통계적으로 의미있는 차이가 보인다.    
      ex) A = 6.19 , B = 9.71 => (9.71/6.19) - 1 = 57% > 35% (MDE)      
    &nbsp;&nbsp;&nbsp;&nbsp;  &nbsp;&nbsp;&nbsp;&nbsp;  # A,B = 전환율   
    &nbsp;&nbsp;&nbsp;&nbsp;  &nbsp;&nbsp;&nbsp;&nbsp;  # MDE(Minumum Detectable Effect) : 두 샘플의 전환율이 의미를 갖는 최소한의 차이   
         
2. 순 방문자 수 확인 : 중복을 제거하여 정확한 결과를 도출한다.
3. 진행기간 : 30일 이하 추천
	- 통계적 측면 : 샘플사이즈 증가 -> 표준오차 감소 -> 그래프가 모수에 가까워지며 P-Value 감소 -> 1종오류 가능성
	- 비즈니스적 측면 : 검증되지 않은 새로운 소재이기 때문에 오래하면 좋지 않음   
![image](https://user-images.githubusercontent.com/110000734/184354871-17b8b584-9216-40df-8831-7771a847e9ea.png)
4.  각 샘플의 전환률
5. 대조군과 실험군의 샘플 수 (옵티마이즐리 = AB-test의 적정 샘플 사이즈 계산 사이트)   

```python
df.isunll().sum()            # 결측값(missing value 확인)

df.shape[0]                  # 데이터 갯수 확인(n>=30)

df.['기본키변수'].nunique     # 순 방문자수 확인

df.duplicated(['유저아이디']) # 중복값 확인

df[df.유저아이디 == "값"]       # 유저아이디 중복 이유 확인

df[(df.유저아이디 == "값1") | (df.유저아이디 == "값2")]  

ab_data_clean = ab_data[(ab_data.그룹 == "experiment") == (ab_data.랜딩페이지 == "new_page")]  # 실험그룹에 new_page 나오는 변수 생성
ab_data_clean[((ab_data_clean['그룹'] == 'experiment') == (ab_data_clean['랜딩페이지'] == 'new_page')) == False].shape[0]

날짜칼럼에서 날짜 빼오기
from datetime import datetime
df = pd.to_datetime(df[‘날짜컬럼’])
df.max() -df.min()            # 소요기간 계산(30일 이하여부)

ctr_pop = df.전환률.mean()                             # 전체 전환률
ctr_control = df.query('그룹 == "컨트롤"').전환률.mean() # 대조군 전환률
ctr_exp = df.query('그룹 == "실험"').전환률.mean()       # 실험군 전환률
ctr_diff = ctr_exp - ctr_control                         # 실험군 - 대조군 전환률

ab_data_clean[ab_data_clean.랜딩페이지 == "new_page"].shape[0] / ab_data_clean.shape[0] # 인구비율 실험 / 전체 
ab_data_clean[ab_data_clean.랜딩페이지 == "old_page"].shape[0] / ab_data_clean.shape[0] # 인구비율 대조 / 전체
```
----------------------------------------
### AB테스트 귀무가설 확인 하는 방법
> Ho = Ha = H(전체 전환률)   
> 두 데이터의 접점을 확인하여 귀무가설 채택 or 기각   
```python
ab_data_clean.클릭.mean()            # 귀무가설의 모수

n_exp = ab_data_clean.query('그룹 == "experiment"').shape[0]  # 실험군의 수
n_control = ab_data_clean.query('그룹 == "control"').shape[0] # 대조군의 수
```
#### 방법1. random.choice을 이용한 귀무가설simulate
대조군 귀무가설   
```python
import random
np.random.seed(10)

old_page_null = []

for i in range(1000):
  old_page_sim = np.random.choice([0,1], n_control, p = [1-ctr_pop, ctr_pop], replace=True)
  old_page_null.append(old_page_sim.mean())
   # p = (확률) # p확률을 가진 모집단에서 샘플링한다는 의미(전체 전환률이 같다고 보기 때문에 전체 전환률 사용)

old_page_null = np.array(old_page_null)
old_page_null_mean = old_page_null.mean() # 대조군의 샘플 전환률
```   
실험군 귀무가설
```python
np.random.seed(10)

new_page_null = []

for i in range(1000):
  new_page_sim = np.random.choice([0,1], n_exp, p = [1-ctr_pop, ctr_pop], replace=True)
  new_page_null.append(new_page_sim.mean())
  
new_page_null = np.array(new_page_null)

new_page_null_mean = new_page_null.mean() # 실험군의 샘플 전환률

```
실험군 평균과 대조군 평균의 차이 귀무가설
```python
np.random.seed(10)

p_diffs = []

for _ in range(1000):
    new_page_converted = np.random.choice([0,1], n_exp, p = [1-ctr_pop, ctr_pop], replace=True)
    old_page_converted = np.random.choice([0,1], n_control, p = [1-ctr_pop, ctr_pop], replace=True)
    p_diffs.append(new_page_converted.mean() - old_page_converted.mean()) 
    
p_diffs = np.array(p_diffs)
 
(p_diffs > ctr_diff).mean()  # p-value 값 모집단의 전환률 > 샘플의 전환률
```
#### 방법2. Binomial distribution
```python
exp_converted_sim = np.random.binomial(n_exp, ctr_pop, 10000)/n_exp
old_converted_sim = np.random.binomial(n_control, ctr_pop, 10000)/n_control

# np.random.binomial(실험군 수, 전체 전환률, 샘플링 수)/실험군 수   
# np.random.binomial(대조군 수, 전체 전환률, 샘플링 수)/대조군 수 

p_diffs = exp_converted_sim - old_converted_sim

(p_diffs > ctr_diff).mean() # p-value 값 모집단의 전환률 > 샘플의 전환률
```
#### 방법3. z-test
```python
import statsmodels.api as sm   

old_convert = ab_data_clean.query('랜딩페이지 == "old_page"').클릭.sum()
new_convert = ab_data_clean.query('랜딩페이지 == "new_page"').클릭.sum()

z_score, p_value = sm.stats.proportions_ztest([old_convert, new_convert], [n_control, n_exp], alternative='smaller')
                                          # ([대조군 전환수,실험군 통계치2,], [대조군 수,실험군 수], alternative =(대립가설))   
```
---
### z-test
- 모집단이 정규분포여야 한다.
- 모분산을 알고 있는 경우(모분산을 몰라도 정규분포면 가능하다.)
- 표본의 크기가 커야한다.(n>=30)
- Z분포는 평균이며, 평균이 0이고 표준편차가 1이다.
- 모집단의 크기가 크기 때문에 부트스트래핑을 통해서 극한중심원리를 바탕으로 샘플의 표준편차를 사용해도 된다.
---

### 오늘의 함수
```python
데이터 지우기
df.drop(변수, inplace = True) #inplace = True(드랍 후 데이터 저장)or False(드랍 후 데이터 저장X)   
df.duplicates['칼럼']


.value_counts(normalize = True) # 값 별 normalize = True(비율저장) or False(갯수저장) , ascending = True(오름)or False(내림)
  ex) per = df['offer'].value_counts(normalize=True) # offer 값 별 칼럼의 비율 추출
      round(per[0] - per[1], 3)                      # 0행의 비율 - 1행의 비율 .000


df.groupby('변수1').변수2.mean()  # 변수1을 기준으로 변수2의 평균
```
