# 202044006 3-A 한상호
노인복지 취약지대 분석

### 한글 폰트 설정
~~~python
!sudo apt-get install -y fonts-nanum
!sudo fc-cache -fv
!rm ~/.cache/matplotlib -rf

import os
os.kill(os.getpid(), 9)
~~~

### 1. 고령화 수준
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



### 2. 고령 인구 절대 규모
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


### 3. 최근 10년간 고령화 증가
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


### 4. 독거노인이 많은 지역
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



### 5. 
