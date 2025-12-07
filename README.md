# 202044006 3-A 한상호
노인복지 취약지대 분석

### 한글 폰트 설정
~~~
!sudo apt-get install -y fonts-nanum
!sudo fc-cache -fv
!rm ~/.cache/matplotlib -rf

import os
os.kill(os.getpid(), 9)
~~~

### 1. 고령화 수준
~~~
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

<img width="1189" height="790" alt="image" src="https://github.com/user-attachments/assets/6f45ee44-3608-4f34-8a84-06d9cfd79ef8" />
