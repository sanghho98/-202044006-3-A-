# 빅데이터 프로젝트
### 이름: 한상호
### 학번: 202044006
### 프로젝트 명: 노인복지 취약지대 분석
### 프로젝트 범위: 데이터 수집, 데이터 가공/정제, 데이터 시각화


## 한글 폰트 설정
~~~python
!sudo apt-get install -y fonts-nanum
!sudo fc-cache -fv
!rm ~/.cache/matplotlib -rf

import os
os.kill(os.getpid(), 9)
~~~

## 1. 고령화 수준
<img width="1189" height="790" alt="image" src="https://github.com/user-attachments/assets/6f45ee44-3608-4f34-8a84-06d9cfd79ef8" />

~~~python
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd


df = pd.read_csv('/content/고령인구비율_시도_시_군_구__2025년.csv', encoding='euc-kr')
df_copy = df.copy()

# 데이터 가공/정제
original_cols = df_copy.columns.tolist()
first_row_values = df_copy.iloc[0].tolist()

new_column_names = []
for i, col_name in enumerate(original_cols):
    if i == 0:
        new_column_names.append('행정구역')
    else:
        # 첫 번째 행 '<br>', ' ', '(', ')', '%' 제거
        descriptive_name = first_row_values[i].replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')

        # 컬럼 이름에서 월 추출
        if '.' in col_name:
            month_part = col_name.split('.')[1]
            new_column_names.append(f'{descriptive_name}_{month_part}')
        else:
            new_column_names.append(descriptive_name)

df_copy.columns = new_column_names
df_copy = df_copy.drop(df_copy.index[0])
df_copy = df_copy.reset_index(drop=True)

# 2025년 11월 컬럼 선택
col_65_plus_pop_nov = None # 65세 이상 인구 컬럼
col_total_pop_nov = None   # 전체 인구 컬럼

for col in df_copy.columns:
    if '65세이상인구' in col and '_11' in col:
        col_65_plus_pop_nov = col
    if '전체인구' in col and '_11' in col:
        col_total_pop_nov = col

# 숫자형으로 변환
df_copy[col_65_plus_pop_nov] = pd.to_numeric(df_copy[col_65_plus_pop_nov], errors='coerce')
df_copy[col_total_pop_nov] = pd.to_numeric(df_copy[col_total_pop_nov], errors='coerce')

# 2025년 11월 고령화율 계산
df_copy['고령화율_2025.11'] = (df_copy[col_65_plus_pop_nov] / df_copy[col_total_pop_nov]) * 100

# 고령화율_2025.11 NaN 행 제거
plot_data = df_copy[df_copy['행정구역'] != '전국'].dropna(subset=['고령화율_2025.11'])

# 고령화율을 기준 정렬
plot_data = plot_data.sort_values(by='고령화율_2025.11', ascending=False)

# 시각화
plt.figure(figsize=(12, 8))
sns.barplot(x='고령화율_2025.11', y='행정구역', data=plot_data, palette='viridis', hue='행정구역', legend=False)
plt.title('2025년 11월 행정구역별 고령화율', fontsize=16)
plt.xlabel('고령화율 (%)', fontsize=12)
plt.ylabel('행정구역', fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()
~~~


<img width="1220" height="727" alt="image" src="https://github.com/user-attachments/assets/4ed275b5-2279-44cb-9b90-27d9eec469b5" />

~~~ python
import pandas as pd
import folium
import json
import os
import unicodedata
from shapely.geometry import shape

df_elderly = pd.read_csv('/content/고령인구비율_시도_시_군_구__2025년.csv', encoding='euc-kr')
df_elderly_processed = df_elderly.copy()

original_cols_elderly = df_elderly_processed.columns.tolist()
first_row_values_elderly = df_elderly_processed.iloc[0].tolist()

# 데이터 가공/정제
new_column_names_elderly = []
for i, col_name in enumerate(original_cols_elderly):
    if i == 0:
        new_column_names_elderly.append('행정구역')
    else:
        # 첫 번째 행의 불필요한 문자 제거 및 정리
        descriptive_name = first_row_values_elderly[i].replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')
        
        # 컬럼 이름에서 연도와 월 부분 추출
        time_part = ''
        if '.' in col_name:
            parts = col_name.split('.')
            if len(parts) == 2:
                time_part = col_name
            elif len(parts) == 3:
                time_part = f"{parts[0]}.{parts[1]}"
        else:
            time_part = col_name

        new_column_names_elderly.append(f'{descriptive_name}_{time_part}')

df_elderly_processed.columns = new_column_names_elderly
df_elderly_processed = df_elderly_processed.drop(df_elderly_processed.index[0])
df_elderly_processed = df_elderly_processed.reset_index(drop=True)

# 2025년 11월 '고령인구비율' 컬럼 선택
col_aging_rate_nov = None
for col in df_elderly_processed.columns:
    if '고령인구비율' in col and '2025.11' in col:
        col_aging_rate_nov = col
        break

# 필요한 컬럼만 선택하여 새 데이터프레임 생성
df_aging_rate_nov = df_elderly_processed[['행정구역', col_aging_rate_nov]].copy()
# 컬럼 이름 변경
df_aging_rate_nov = df_aging_rate_nov.rename(columns={
    col_aging_rate_nov: '고령화율_2025.11'
})

# '고령화율_2025.11' 컬럼을 숫자형으로 변환 (변환 실패 시 NaN으로 처리)
df_aging_rate_nov['고령화율_2025.11'] = pd.to_numeric(df_aging_rate_nov['고령화율_2025.11'], errors='coerce')

# '전국' 데이터를 제외하고, NaN 값이 있는 행 제거
df_aging_rate_nov = df_aging_rate_nov[df_aging_rate_nov['행정구역'] != '전국'].dropna(subset=['고령화율_2025.11'])


# 대한민국 행정구역 GeoJSON 로드 및 전처리
geojson_file_path = '/content/skorea-provinces-geo.json'

# GeoJSON 영문 시도명을 '행정구역' 데이터프레임의 한글 시도명 매핑
# 수동 매핑
province_name_mapping = {
    'Busan': '부산광역시',
    'Chungcheongbuk-do': '충청북도',
    'Chungcheongnam-do': '충청남도',
    'Daegu': '대구광역시',
    'Daejeon': '대전광역시',
    'Gangwon-do': '강원특별자치도',
    'Gwangju': '광주광역시',
    'Gyeonggi-do': '경기도',
    'Gyeongsangbuk-do': '경상북도',
    'Gyeongsangnam-do': '경상남도',
    'Incheon': '인천광역시',
    'Jeju': '제주특별자치도',
    'Jeollabuk-do': '전북특별자치도',
    'Jeollanam-do': '전라남도',
    'Sejong': '세종특별자치시',
    'Seoul': '서울특별시',
    'Ulsan': '울산광역시',
}

for feature in geo_data['features']:
    english_name = feature['properties']['NAME_1']
    feature['properties']['KOR_NAME'] = province_name_mapping.get(english_name, english_name)

# 고령화율 데이터와 GeoJSON 병합
geojson_names = set(feature['properties']['KOR_NAME'] for feature in geo_data['features'])
df_names = set(df_aging_rate_nov['행정구역'].unique())

missing_in_df = geojson_names - df_names
missing_in_geojson = df_names - geojson_names

if missing_in_df:
    print(f"GeoJSON에 있으나 DataFrame에 없는 행정구역: {missing_in_df}")
if missing_in_geojson:
    print(f"DataFrame에 있으나 GeoJSON에 없는 행정구역: {missing_in_geojson}")

df_aging_rate_nov = df_aging_rate_nov[df_aging_rate_nov['행정구역'].isin(geojson_names)].copy()


# Folium Choropleth 지도 생성

# 대한민국 중심 좌표 설정 (서울)
latitude = 37.5665
longitude = 126.9780

m = folium.Map(location=[latitude, longitude], zoom_start=7, tiles='cartodbpositron')

folium.Choropleth(
    geo_data=geo_data,
    data=df_aging_rate_nov,
    columns=['행정구역', '고령화율_2025.11'],
    key_on='feature.properties.KOR_NAME',
    fill_color='YlGnBu',
    fill_opacity=0.7,
    line_opacity=0.2,
    legend_name='2025년 11월 고령화율 (%)',
    highlight=True,
    line_color='black',
).add_to(m)

# 맵에 툴팁 추가
folium.features.GeoJson(
    geo_data,
    name='행정구역',
    style_function=lambda x: {'color': 'black', 'fillColor': 'transparent', 'weight': 0.5},
    tooltip=folium.features.GeoJsonTooltip(fields=['KOR_NAME'], aliases=['행정구역'], localize=True) # 툴팁에 한글 시도명 표시
).add_to(m)

# 레이블 위치 조정 (겹침 방지 목적)
label_offsets = {
    '인천광역시': (0.02, -0.1),
    '서울특별시': (-0.02, 0.05),
    '경기도': (0.08, 0),
    '세종특별자치시': (0.0, 0.05)
}

# 지리적 중심점 계산 및 행정구역 이름 레이블 추가
for feature in geo_data['features']:
    kor_name = feature['properties']['KOR_NAME']
    
    if kor_name in df_aging_rate_nov['행정구역'].values:
        geo_shape = shape(feature['geometry'])
        centroid = geo_shape.centroid
        
        offset_lat, offset_lon = label_offsets.get(kor_name, (0, 0))
        label_location = [centroid.y + offset_lat, centroid.x + offset_lon]

        folium.Marker(
            location=label_location,
            icon=folium.DivIcon(
                icon_size=(150, 20),
                icon_anchor=(75, 10),
                html=f"<div style='font-size: 10pt; font-weight: bold; color : black; text-align: center; width: 150px;'>{kor_name}</div>"
            )
        ).add_to(m)

# 지도 표시
m
~~~



## 2. 고령 인구 절대 규모
<img width="1188" height="790" alt="image" src="https://github.com/user-attachments/assets/c3903baf-0f0d-44f1-96c1-1e195675c1ad" />

~~~ python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import os
import unicodedata
import matplotlib.ticker as mticker

df_elderly = pd.read_csv('/content/고령인구비율_시도_시_군_구__2025년.csv', encoding='euc-kr')
df_elderly_processed = df_elderly.copy()

original_cols_elderly = df_elderly_processed.columns.tolist()
first_row_values_elderly = df_elderly_processed.iloc[0].tolist()

# 데이터 가공/정제
new_column_names_elderly = []
for i, col_name in enumerate(original_cols_elderly):
    if i == 0:
        new_column_names_elderly.append('행정구역')
    else:
        # 첫 번째 행의 불필요한 문자 제거 및 정리
        descriptive_name = first_row_values_elderly[i].replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')

        # 컬럼 이름에서 연도와 월 부분 추출
        time_part = ''
        if '.' in col_name:
            parts = col_name.split('.')
            if len(parts) == 2:
                time_part = col_name
            elif len(parts) == 3:
                time_part = f"{parts[0]}.{parts[1]}"
        else:
            time_part = col_name

        new_column_names_elderly.append(f'{descriptive_name}_{time_part}') # '설명_YYYY.MM' 형태로 새 이름 생성

df_elderly_processed.columns = new_column_names_elderly
df_elderly_processed = df_elderly_processed.drop(df_elderly_processed.index[0])
df_elderly_processed = df_elderly_processed.reset_index(drop=True)

# 2025년 11월 '65세이상인구' 컬럼 선택
col_65_plus_pop_nov = None
for col in df_elderly_processed.columns:
    if '65세이상인구' in col and '2025.11' in col:
        col_65_plus_pop_nov = col
        break

plot_data_abs_elderly = df_elderly_processed[['행정구역', col_65_plus_pop_nov]].copy()
plot_data_abs_elderly = plot_data_abs_elderly.rename(columns={
    col_65_plus_pop_nov: '65세이상인구_2025.11'
})

# '65세이상인구_2025.11' 컬럼을 숫자형으로 변환 (변환 실패 시 NaN으로 처리)
plot_data_abs_elderly['65세이상인구_2025.11'] = pd.to_numeric(plot_data_abs_elderly['65세이상인구_2025.11'], errors='coerce')

# '전국' 데이터를 제외하고, NaN 값이 있는 행 제거
plot_data_abs_elderly = plot_data_abs_elderly[plot_data_abs_elderly['행정구역'] != '전국'].dropna(subset=['65세이상인구_2025.11'])

# '65세이상인구_2025.11'을 기준으로 정렬
plot_data_abs_elderly = plot_data_abs_elderly.sort_values(by='65세이상인구_2025.11', ascending=False)

# 시각화

plt.figure(figsize=(12, 8))
sns.barplot(
    x='65세이상인구_2025.11',
    y='행정구역',
    data=plot_data_abs_elderly,
    palette='magma',
    hue='행정구역',
    legend=False
)

ax = plt.gca()
formatter = mticker.FuncFormatter(lambda x, p: format(int(x), ','))
ax.xaxis.set_major_formatter(formatter)

plt.title('2025년 11월 행정구역별 65세 이상 고령인구', fontsize=16)
plt.xlabel('65세 이상 고령인구 (명)', fontsize=12)
plt.ylabel('행정구역', fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()
~~~


## 3. 최근 10년간 고령화 증가
<img width="1492" height="989" alt="image" src="https://github.com/user-attachments/assets/c217801c-4d7c-4567-b11a-3ff16d827dde" />

~~~ python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import re
import matplotlib.dates as mdates

df = pd.read_csv('/content/고령인구비율_시도_시_군_구__년도별 10년.csv', encoding='euc-kr')

# 데이터 가공/정제
df_copy = df.copy()
original_cols_annual = df_copy.columns.tolist()
first_row_values_annual = df_copy.iloc[0].tolist()

new_column_names_annual = []
for i, col_name in enumerate(original_cols_annual):
    if i == 0:
        new_column_names_annual.append('행정구역') 
    else:
        # 첫 번째 행 '<br>', ' ', '(', ')', '%' 제거
        descriptive_name = first_row_values_annual[i].replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')

        # 컬럼 이름에서 연도 및 월 추출
        time_part = ''
        if '.' in col_name:
            parts = col_name.split('.')
            if len(parts) == 2: 
                time_part = col_name
            elif len(parts) == 3: 
                time_part = f"{parts[0]}.{parts[1]}"
        else: 
            time_part = col_name

        new_column_names_annual.append(f'{descriptive_name}_{time_part}')

df_copy.columns = new_column_names_annual 
df_copy = df_copy.drop(df_copy.index[0]) 
df_copy = df_copy.reset_index(drop=True) 

df_melted = df_copy.melt(id_vars=['행정구역'], var_name='Metric_Year_Month', value_name='Value')

# 지표 유형과 연/월 분리
def parse_metric_year_month(col_str):
    parts = col_str.split('_')
    metric_type = '_'.join(parts[:-1]) 
    raw_year_month = parts[-1] 

    # 연/월 부분 형식 변환
    if '.' in raw_year_month:
        ym_parts = raw_year_month.split('.')
        if len(ym_parts[1]) == 2: 
            year_month = f"{ym_parts[0]}.{ym_parts[1]}"
        else: 
            year_month = ym_parts[0]
    else: 
        year_month = raw_year_month

    return metric_type, year_month

df_melted[['Metric_Type', 'Year_Month']] = df_melted['Metric_Year_Month'].apply(lambda x: pd.Series(parse_metric_year_month(x)))
df_melted = df_melted.drop(columns=['Metric_Year_Month']) 
df_melted = df_melted[df_melted['Metric_Type'] != '고령인구비율_A÷B×100'] 
df_melted['Value'] = pd.to_numeric(df_melted['Value'], errors='coerce') 


df_pivot = df_melted.pivot_table(index=['행정구역', 'Year_Month'], columns='Metric_Type', values='Value').reset_index()
df_pivot.columns.name = None 
df_pivot = df_pivot.rename(columns={
    '65세이상인구_A_명': '65세이상인구',
    '전체인구_B_명': '전체인구'
}) 
df_pivot = df_pivot.dropna(subset=['65세이상인구', '전체인구']) # 필수 컬럼에 NaN 행 제거

# 고령화율 계산
df_pivot['고령화율'] = (df_pivot['65세이상인구'] / df_pivot['전체인구']) * 100 

df_pivot['Year_Month_standardized'] = df_pivot['Year_Month'].apply(lambda x: x if '.' in x else f'{x}.01')
df_pivot['Year_Month_dt'] = pd.to_datetime(df_pivot['Year_Month_standardized'], format='%Y.%m', errors='coerce')
df_pivot = df_pivot.dropna(subset=['Year_Month_dt']) 

df_pivot = df_pivot.sort_values(by=['행정구역', 'Year_Month_dt']).reset_index(drop=True) # 행정구역 및 날짜 기준으로 정렬
plot_data_final = df_pivot[df_pivot['행정구역'] != '전국'].copy() # '전국' 데이터 제외

# 시각화
plt.figure(figsize=(15, 10)) 
sns.lineplot(x='Year_Month_dt', y='고령화율', hue='행정구역', data=plot_data_final, marker='o')

plt.title('행정구역별 고령화율 추이 10년간 (2015-2025.11)', fontsize=16) 
plt.xlabel('연도', fontsize=12) 
plt.ylabel('고령화율 (%)', fontsize=12) 

ax = plt.gca() 
ax.xaxis.set_major_locator(mdates.YearLocator()) 
ax.xaxis.set_major_formatter(mdates.DateFormatter('%Y')) 
plt.xticks(rotation=45) 

plt.grid(True, linestyle='--', alpha=0.7) 
plt.legend(title='행정구역', bbox_to_anchor=(1.05, 1), loc='upper left') 
plt.tight_layout() 
plt.show()
~~~

<img width="1389" height="889" alt="image" src="https://github.com/user-attachments/assets/9f0486b4-b541-4e9c-b303-fb49f2eef5ad" />

~~~ python
import matplotlib.pyplot as plt
import seaborn as sns
import pandas as pd

plt.figure(figsize=(14, 9))

# Sort the comparison_df by '고령화율_증가량' for better visualization
sorted_comparison_df = comparison_df.sort_values(by='고령화율_증가량', ascending=False)

sns.barplot(x='고령화율_증가량', y='행정구역', data=sorted_comparison_df, palette='viridis', hue='행정구역', legend=False)
plt.title('2015년 ~ 2025년 11월 행정구역별 고령화율 증가량', fontsize=18)
plt.xlabel('고령화율 증가량 (%, 2015년 대비 2025년 11월)', fontsize=14)
plt.ylabel('행정구역', fontsize=14)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()
~~~


## 4. 독거노인이 많은 지역
<img width="1190" height="790" alt="image" src="https://github.com/user-attachments/assets/0ec1861f-97ff-4bf8-b4f9-b74b1fc81562" />

~~~ python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import os 
import unicodedata 

# CSV 파일 로드
df_single_elderly = pd.read_csv('/content/독거노인가구비율_시도_시_군_구__20251205121312.csv', encoding='euc-kr')
df_se_processed = df_single_elderly.copy()

original_cols_se = df_se_processed.columns.tolist()
first_row_values_se = df_se_processed.iloc[0].tolist()

# 데이터 가공/정제
new_column_names_se = []
for i, col_name in enumerate(original_cols_se):
    if i == 0:
        new_column_names_se.append('행정구역')
    else:
        descriptive_name = first_row_values_se[i].replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')

        # 컬럼 이름에서 연도 추출
        year_part = ''
        if '.' in col_name:
            year_part = col_name.split('.')[0]
        else:
            year_part = col_name
        new_column_names_se.append(f'{descriptive_name}_{year_part}')

df_se_processed.columns = new_column_names_se
df_se_processed = df_se_processed.drop(df_se_processed.index[0])
df_se_processed = df_se_processed.reset_index(drop=True)

# 2024년 독거노인비율 컬럼 선택
col_single_elderly_ratio_2024 = None
for col in df_se_processed.columns:
    if '독거노인가구비율' in col and '_2024' in col:
        col_single_elderly_ratio_2024 = col
        break

df_se_processed[col_single_elderly_ratio_2024] = pd.to_numeric(df_se_processed[col_single_elderly_ratio_2024], errors='coerce')

# '전국' 데이터를 필터링, NaN 행 삭제
plot_data_se = df_se_processed[df_se_processed['행정구역'] != '전국'].dropna(subset=[col_single_elderly_ratio_2024])

# 비율 기준 정렬
plot_data_se = plot_data_se.sort_values(by=col_single_elderly_ratio_2024, ascending=False)

# 시각화
plt.figure(figsize=(12, 8))
sns.barplot(x=col_single_elderly_ratio_2024, y='행정구역', data=plot_data_se, palette='viridis', hue='행정구역', legend=False)
plt.title('2024년 행정구역별 독거노인가구비율', fontsize=16)
plt.xlabel('독거노인가구비율 (%)', fontsize=12)
plt.ylabel('행정구역', fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()
~~~



## 5. 고령화율-기초생활수급자 비율 (4분면 분석)
<img width="1388" height="989" alt="image" src="https://github.com/user-attachments/assets/a9fa88e1-d26b-4f3b-9829-6a33898702cd" />

~~~ python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import os
import unicodedata
import matplotlib.font_manager as fm

# 고령화율
df_elderly = pd.read_csv('/content/고령인구비율_시도_시_군_구__2025년.csv', encoding='euc-kr')
df_elderly_processed = df_elderly.copy()

original_cols_elderly = df_elderly_processed.columns.tolist()
first_row_values_elderly = df_elderly_processed.iloc[0].tolist()

# 데이터 가공/정제
new_column_names_elderly = []
for i, col_name in enumerate(original_cols_elderly):
    if i == 0:
        new_column_names_elderly.append('행정구역') 
    else:
        # 첫 번째 행 불필요한 문자 제거 및 정리
        descriptive_name = first_row_values_elderly[i].replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')
        
        # 컬럼 이름에서 연도와 월 부분 추출
        time_part = ''
        if '.' in col_name:
            parts = col_name.split('.')
            if len(parts) == 2:
                time_part = col_name
            elif len(parts) == 3:
                time_part = f"{parts[0]}.{parts[1]}"
        else:
            time_part = col_name

        new_column_names_elderly.append(f'{descriptive_name}_{time_part}')

df_elderly_processed.columns = new_column_names_elderly
df_elderly_processed = df_elderly_processed.drop(df_elderly_processed.index[0])
df_elderly_processed = df_elderly_processed.reset_index(drop=True)

col_65_plus_pop_oct = None
col_total_pop_oct = None

for col in df_elderly_processed.columns:
    if '65세이상인구' in col and '2025.10' in col:
        col_65_plus_pop_oct = col
    if '전체인구' in col and '2025.10' in col:
        col_total_pop_oct = col

if not col_65_plus_pop_oct or not col_total_pop_oct:
    raise ValueError("'고령인구비율' 파일에서 2025년 10월 '65세이상인구' 또는 '전체인구' 컬럼을 찾을 수 없습니다.")

# 필요한 컬럼만 선택하여 새 데이터프레임 생성
df_elderly_oct = df_elderly_processed[['행정구역', col_65_plus_pop_oct, col_total_pop_oct]].copy()
# 컬럼 이름 변경
df_elderly_oct = df_elderly_oct.rename(columns={
    col_65_plus_pop_oct: '65세이상인구',
    col_total_pop_oct: '전체인구'
})

# '65세이상인구'와 '전체인구' 컬럼을 숫자형으로 변환 (변환 실패 시 NaN으로 처리)
df_elderly_oct['65세이상인구'] = pd.to_numeric(df_elderly_oct['65세이상인구'], errors='coerce')
df_elderly_oct['전체인구'] = pd.to_numeric(df_elderly_oct['전체인구'], errors='coerce')

# '전국' 데이터를 제외하고, NaN 값이 있는 행은 제거
df_elderly_oct = df_elderly_oct[df_elderly_oct['행정구역'] != '전국'].dropna(subset=['65세이상인구', '전체인구'])

# 2025년 10월 고령화율 계산
df_elderly_oct['고령화율_2025.10'] = (df_elderly_oct['65세이상인구'] / df_elderly_oct['전체인구']) * 100


# 기초생활보장 수급자수
df_social_welfare = pd.read_csv('/content/기초생활보장 연령별 수급자수 10월.csv', encoding='euc-kr')

# 2025년 10월 데이터와 65세 이상 연령대 필터링
df_filtered_sw = df_social_welfare[
    (df_social_welfare['통계연월'] == 202510) &
    (df_social_welfare['연령'] >= 65)
].copy()

# '통계시도명'별로 '수급자수' 집계
df_aggregated_sw = df_filtered_sw.groupby('통계시도명')['수급자수'].sum().reset_index()
# 컬럼 이름 변경
df_aggregated_sw = df_aggregated_sw.rename(columns={
    '통계시도명': '행정구역',
    '수급자수': '65세 이상 수급자수'
})


# 두 데이터프레임 병합 및 비율 계산

# '행정구역' 컬럼을 기준으로 두 데이터프레임을 내부 조인
merged_df = pd.merge(df_elderly_oct, df_aggregated_sw, on='행정구역', how='inner')

# 65세 이상 노인 인구 중 기초생활보장 수급자 비율 계산
merged_df['65세 이상 기초생활수급자 비율'] = (merged_df['65세 이상 수급자수'] / merged_df['65세이상인구']) * 100

# '전국' 데이터는 지역별 비교에서 제외하고, 계산된 비율이 NaN인 행은 제거
plot_data_scatter = merged_df[merged_df['행정구역'] != '전국'].dropna(subset=['고령화율_2025.10', '65세 이상 기초생활수급자 비율'])


# 4분면 산점도 시각화

plt.figure(figsize=(14, 10))

# 산점도 생성
sns.scatterplot(
    x='고령화율_2025.10', 
    y='65세 이상 기초생활수급자 비율', 
    data=plot_data_scatter, 
    hue='행정구역',
    s=150,
    alpha=0.8,
    legend='full'
)

for i, row in plot_data_scatter.iterrows():
    plt.text(
        row['고령화율_2025.10'], 
        row['65세 이상 기초생활수급자 비율'], 
        row['행정구역'], 
        fontsize=9, 
        ha='right', 
        va='bottom', 
        color='black' 
    )

# 4분면 구분을 위한 평균선(기준선)
mean_aging_rate = plot_data_scatter['고령화율_2025.10'].mean()
mean_recipient_ratio = plot_data_scatter['65세 이상 기초생활수급자 비율'].mean()

plt.axvline(x=mean_aging_rate, color='r', linestyle='--', linewidth=1.5, label=f'평균 고령화율: {mean_aging_rate:.2f}%')
plt.axhline(y=mean_recipient_ratio, color='b', linestyle='--', linewidth=1.5, label=f'평균 수급자 비율: {mean_recipient_ratio:.2f}%')

plt.title('2025년 10월 행정구역별 고령화율 vs 65세 이상 기초생활수급자 비율 (4분면 분석)', fontsize=16) 
plt.xlabel('고령화율 (%)', fontsize=12)
plt.ylabel('65세 이상 기초생활수급자 비율 (%)', fontsize=12)
plt.grid(True, linestyle=':', alpha=0.6)
plt.legend(title='행정구역 및 평균', bbox_to_anchor=(1.05, 1), loc='upper left')
plt.tight_layout()
plt.show()
~~~



## 6. 노인 1000명당 돌봄 공급 수준
### 6.1 재가방문돌봄
<img width="1189" height="790" alt="image" src="https://github.com/user-attachments/assets/6c4c3c08-a9cf-4324-81e5-09cf5919ff17" />

### 6.2 중간돌봄(주 단기 보호시설)
<img width="1189" height="790" alt="image" src="https://github.com/user-attachments/assets/7e976f8f-b57f-40cd-ba88-097ba95c0a15" />

### 6.3 시설돌봄(요양시설)
<img width="1189" height="790" alt="image" src="https://github.com/user-attachments/assets/b072d782-2f80-462c-b30d-b3050c15d591" />


~~~ python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import os
import unicodedata
import matplotlib.ticker as mticker
import matplotlib.font_manager as fm

# '장기요양기관 현황' 데이터 로드 및 전처리
df_ltc = pd.read_csv('/content/연도별_시·도별_급여종류별_장기요양기관_현황_20251206234402.csv', encoding='euc-kr', header=None)

# 데이터 가공/정제
new_columns = []
for i, col_name_lvl0 in enumerate(df_ltc.iloc[0]):
    col_name_lvl1 = df_ltc.iloc[1, i]
    
    if '시도별' in str(col_name_lvl0):
        new_columns.append('행정구역')
    elif '급여종류별' in str(col_name_lvl0):
        new_columns.append('급여종류')
    else:
        year_part = str(col_name_lvl0)
        metric_part = str(col_name_lvl1).replace(' (개)', '').replace(' (명)', '').strip()
        new_columns.append(f'{year_part}_{metric_part}')

df_ltc_processed = df_ltc.copy()
df_ltc_processed.columns = new_columns
df_ltc_processed = df_ltc_processed.iloc[2:].reset_index(drop=True)

# 2024년 '기관수' 데이터 추출
df_ltc_2024_institutions = df_ltc_processed[[ 
    '행정구역', 
    '급여종류'] + [col for col in df_ltc_processed.columns if col.startswith('2024_기관수')
]].copy()

# '2024_기관수' 컬럼 이름 '기관수'로 변경
df_ltc_2024_institutions = df_ltc_2024_institutions.rename(columns={'2024_기관수': '기관수'})

# '합계' 행을 제거, 전체 서비스 종류 합계 행 제거
df_ltc_2024_institutions = df_ltc_2024_institutions[
    (df_ltc_2024_institutions['행정구역'] != '합계') & 
    (df_ltc_2024_institutions['급여종류'] != '계')
].copy()

# '기관수' 컬럼을 숫자형으로 변환 (변환 실패 시 NaN으로 처리 후 0으로 채움)
df_ltc_2024_institutions['기관수'] = pd.to_numeric(df_ltc_2024_institutions['기관수'], errors='coerce').fillna(0)

# '급여종류'를 컬럼 생성, '기관수' 값
df_ltc_pivot = df_ltc_2024_institutions.pivot_table(
    index='행정구역',
    columns='급여종류',
    values='기관수',
    aggfunc='sum'
).reset_index()
df_ltc_pivot.columns.name = None

# 서비스 유형별로 기관수 합산
# 기본값 0 설정
df_ltc_pivot['재가방문돌봄 서비스'] = df_ltc_pivot.get('방문요양', 0) + df_ltc_pivot.get('방문간호', 0) + \
                                    df_ltc_pivot.get('방문목욕', 0) + df_ltc_pivot.get('복지용구', 0) + \
                                    df_ltc_pivot.get('통합재가', 0)
df_ltc_pivot['중간돌봄 시설'] = df_ltc_pivot.get('주야간보호', 0) + df_ltc_pivot.get('단기보호', 0)
df_ltc_pivot['시설돌봄'] = df_ltc_pivot.get('노인요양시설', 0) + df_ltc_pivot.get('노인요양공동생활가정', 0) # 원본 데이터에 맞춰 '노인요양공동생활가정'으로 변경


# '고령인구비율' 데이터 로드 및 전처리
df_elderly_yearly = pd.read_csv('/content/고령인구비율_시도_시_군_구__년도별.csv', encoding='euc-kr', header=None)

new_columns_elderly_yearly = []
for i, col_name_lvl0 in enumerate(df_elderly_yearly.iloc[0]):
    col_name_lvl1 = df_elderly_yearly.iloc[1, i]

    if '행정구역별' in str(col_name_lvl0):
        new_columns_elderly_yearly.append('행정구역')
    else:
        year_month_part = str(col_name_lvl0)
        metric_part = str(col_name_lvl1).replace('<br>', '_').replace(' ', '_').replace('(', '').replace(')', '').replace('%', '').replace('__', '_').strip('_')
        new_columns_elderly_yearly.append(f'{metric_part}_{year_month_part}')

df_elderly_yearly_processed = df_elderly_yearly.copy()
df_elderly_yearly_processed.columns = new_columns_elderly_yearly
df_elderly_yearly_processed = df_elderly_yearly_processed.iloc[2:].reset_index(drop=True)

# 2024년의 65세 이상 인구 데이터 추출
col_65_plus_pop_2024 = None
for col in df_elderly_yearly_processed.columns:
    if '65세이상인구' in col and '2024' in col: # 2024년도 65세 이상 인구 컬럼 검색
        col_65_plus_pop_2024 = col
        break

if not col_65_plus_pop_2024:
    for col in df_elderly_yearly_processed.columns:
        if '65세이상인구' in col and '2025.11' in col:
            col_65_plus_pop_2024 = col
            print("error")
            break

if not col_65_plus_pop_2024:
    raise ValueError("error")

df_elderly_pop = df_elderly_yearly_processed[['행정구역', col_65_plus_pop_2024]].copy()
df_elderly_pop = df_elderly_pop.rename(columns={col_65_plus_pop_2024: '65세이상인구_대상년도'})
df_elderly_pop['65세이상인구_대상년도'] = pd.to_numeric(df_elderly_pop['65세이상인구_대상년도'], errors='coerce')
df_elderly_pop = df_elderly_pop[df_elderly_pop['행정구역'] != '전국'].dropna(subset=['65세이상인구_대상년도']).copy()


# 두 데이터프레임 병합 및 비율 계산
merged_df = pd.merge(df_ltc_pivot, df_elderly_pop, on='행정구역', how='inner')

# 65세 이상 노인 1000명당 장기요양기관 수 계산
merged_df['재가방문돌봄 서비스 (1000명당)'] = (merged_df['재가방문돌봄 서비스'] / merged_df['65세이상인구_대상년도']) * 1000
merged_df['중간돌봄 시설 (1000명당)'] = (merged_df['중간돌봄 시설'] / merged_df['65세이상인구_대상년도']) * 1000
merged_df['시설돌봄 (1000명당)'] = (merged_df['시설돌봄'] / merged_df['65세이상인구_대상년도']) * 1000

plot_data = merged_df[['행정구역',
                       '재가방문돌봄 서비스 (1000명당)',
                       '중간돌봄 시설 (1000명당)',
                       '시설돌봄 (1000명당)']].copy()

# 시각화
# 1. 재가방문돌봄 서비스 수
plt.figure(figsize=(12, 8))
sns.barplot(
    x='재가방문돌봄 서비스 (1000명당)',
    y='행정구역',
    data=plot_data.sort_values(by='재가방문돌봄 서비스 (1000명당)', ascending=False),
    palette='viridis',
    hue='행정구역',
    legend=False
)
plt.title('2024년 행정구역별 65세 이상 노인 1000명당 재가방문돌봄 서비스 수', fontsize=16)
plt.xlabel('재가방문돌봄 서비스 수 (1000명당)', fontsize=12)
plt.ylabel('행정구역', fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()

# 2. 중간돌봄 시설 수
plt.figure(figsize=(12, 8))
sns.barplot(
    x='중간돌봄 시설 (1000명당)',
    y='행정구역',
    data=plot_data.sort_values(by='중간돌봄 시설 (1000명당)', ascending=False),
    palette='magma',
    hue='행정구역',
    legend=False
)
plt.title('2024년 행정구역별 65세 이상 노인 1000명당 중간돌봄 시설 수', fontsize=16)
plt.xlabel('중간돌봄 시설 수 (1000명당)', fontsize=12)
plt.ylabel('행정구역', fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()

# 3. 시설돌봄 수
plt.figure(figsize=(12, 8))
sns.barplot(
    x='시설돌봄 (1000명당)',
    y='행정구역',
    data=plot_data.sort_values(by='시설돌봄 (1000명당)', ascending=False),
    palette='cividis',
    hue='행정구역',
    legend=False
)
plt.title('2024년 행정구역별 65세 이상 노인 1000명당 시설돌봄 수', fontsize=16)
plt.xlabel('시설돌봄 수 (1000명당)', fontsize=12)
plt.ylabel('행정구역', fontsize=12)
plt.grid(axis='x', linestyle='--', alpha=0.7)
plt.tight_layout()
plt.show()
~~~


