import streamlit as st
import pandas as pd
from datetime import datetime, timedelta
import math

# 登录系统
if 'logged_in' not in st.session_state:
    st.session_state.logged_in = False

if not st.session_state.logged_in:
    st.title("养鸡场数据分析系统")
    with st.form("login_form"):
        username = st.text_input("用户名")
        password = st.text_input("密码", type="password")
        if st.form_submit_button("登录"):
            if username == "admin" and password == "123456":
                st.session_state.logged_in = True
                st.rerun()
            else: 
                st.error("用户名或密码错误")
    st.stop()

with st.sidebar:
    if st.button("退出登录"):
        st.session_state.logged_in = False
        st.rerun()

# 导航菜单
st.sidebar.title("分析菜单")
page = st.sidebar.radio("选择功能", ["生产性能", "体重分析", "饲料预测", "进料量分析", "每日分析"])

file_path = r'C:\Users\hb\Desktop\原始数据\chicken.xlsx'

# 生产性能分析
if page == "生产性能":
    st.title("📈 生产性能分析")
    
    def analyze_performance(target_age):
        weight_df = pd.read_excel(file_path, sheet_name='称重数据')
        available_ages = sorted(weight_df[weight_df['日龄'] <= target_age]['日龄'].unique())
        all_results = []
        
        for age in available_ages:
            result = []
            for i in range(1, 17):
                try:
                    df = pd.read_excel(file_path, sheet_name=str(i))
                    target_data = df[df.iloc[:, 2] <= age]
                    total_feed = target_data.iloc[:, 3].sum()
                    total_dead = target_data.iloc[:, 4].sum() + target_data.iloc[:, 5].sum()
                    result.append([f'鸡舍{i}', total_feed, total_dead, age])
                except: continue
            
            weight_target = weight_df[weight_df['日龄'] == age].groupby('鸡舍编号')['均重(g)'].mean().reset_index()
            weight_dict = dict(zip(weight_target['鸡舍编号'], weight_target['均重(g)'].round(0).astype(int)))
            std_df = pd.read_excel(file_path, sheet_name='采食标准')
            std_target = std_df[std_df['日龄'] == age].iloc[0]
            std_feed = std_target['单只鸡累计采食量/克'] * 54
            std_dead_rate = std_target['标准累计死淘%']
            
            comparison = pd.DataFrame(result, columns=['鸡舍', '累计采食(kg)', '累计死淘(只)', '日龄'])
            comparison['采食差异(kg)'] = (comparison['累计采食(kg)'] - std_feed).round(0).astype(int)
            comparison['成活率%'] = ((54000 - comparison['累计死淘(只)']) / 54000 * 100).round(1)
            comparison['与标准死淘差%'] = (comparison['累计死淘(只)'] / 54000 * 100 - std_dead_rate).round(3)
            comparison['周末体重(g)'] = comparison['鸡舍'].apply(lambda x: weight_dict.get(int(x.replace('鸡舍', '')), 0))
            total_live_weight = (54000 - comparison['累计死淘(只)']) * comparison['周末体重(g)'] / 1000
            comparison['料肉比'] = (comparison['累计采食(kg)'] / total_live_weight).round(2)
            comparison = comparison[comparison['周末体重(g)'] > 0]
            all_results.append(comparison)
        
        return pd.concat(all_results, ignore_index=True) if all_results else None

    target_age = st.sidebar.slider("选择分析日龄", 1, 41, 15)
    
    if st.sidebar.button("开始分析"):
        with st.spinner("分析中..."):
            results_df = analyze_performance(target_age)
        
        if results_df is not None:
            for age in sorted(results_df['日龄'].unique()):
                age_data = results_df[results_df['日龄'] == age]
                std_df = pd.read_excel(file_path, sheet_name='采食标准')
                std_feed = std_df[std_df['日龄'] == age].iloc[0]['单只鸡累计采食量/克'] * 54
                std_dead_rate = std_df[std_df['日龄'] == age].iloc[0]['标准累计死淘%']
                
                st.subheader(f"🎯 {age}日龄料肉比分析")
                st.write(f"**标准参考**: 累计采食 {std_feed:.0f}kg, 累计死淘率 {std_dead_rate}%")
                st.write("---")
                
                table_data = []
                for _, row in age_data.iterrows():
                    # 采食量保留具体数字
                    if row['采食差异(kg)'] > 200:
                        feed_status = f"超标准{row['采食差异(kg)']}kg"
                    elif row['采食差异(kg)'] < -200:
                        feed_status = f"低于标准{-row['采食差异(kg)']}kg"
                    else:
                        feed_status = "正常"
                    
                    # 死淘率改为文字描述
                    if row['与标准死淘差%'] > 0.2:
                        dead_status = "高于标准"
                    elif row['与标准死淘差%'] < -0.1:
                        dead_status = "正常"
                    else:
                        dead_status = "正常"
                    
                    # 料肉比颜色标识
                    fcr_color = "🔴" if row['料肉比'] > 1.5 else "🟢" if row['料肉比'] < 1.2 else ""
                    
                    table_data.append([
                        row['鸡舍'],  # 直接使用"鸡舍1"格式
                        f"{row['累计采食(kg)']}kg",
                        feed_status,
                        f"{row['累计死淘(只)']}只",
                        f"{row['成活率%']}%",
                        dead_status,
                        f"{row['周末体重(g)']}g",
                        f"{fcr_color}{row['料肉比']}"
                    ])
                
                df_table = pd.DataFrame(table_data, 
                    columns=['鸡舍', '累计采食', '采食状态', '累计死淘', '成活率', '死淘状态', '体重', '料肉比'])
                # 隐藏索引
                st.table(df_table.set_index('鸡舍'))
                st.write("")

# 体重分析
elif page == "体重分析":
    st.title("⚖️ 体重均匀度分析")
    
    def analyze_weight():
        weight_df = pd.read_excel(file_path, sheet_name='称重数据')
        # 位置映射
        position_map = {15: '前', 30: '中前', 35: '中', 45: '中后', 60: '后'}
        
        for age in sorted(weight_df['日龄'].unique()):
            age_data = weight_df[weight_df['日龄'] == age]
            st.subheader(f"🎯 {age}日龄体重分析")
            st.write("---")
            
            table_data = []
            # 第一轮：收集所有料肉比值
            all_fcr_values = []
            fcr_data = {}
            
            for house in sorted(age_data['鸡舍编号'].unique()):
                house_data = age_data[age_data['鸡舍编号'] == house]
                valid_data = house_data['均重(g)'].dropna()
                if len(valid_data) == 0: continue
                
                avg_weight = int(valid_data.mean())
                
                # 计算料肉比
                try:
                    feed_df = pd.read_excel(file_path, sheet_name=str(house))
                    cumulative_feed = feed_df[feed_df.iloc[:, 2] <= age].iloc[:, 3].sum()
                    
                    dead_df = pd.read_excel(file_path, sheet_name=str(house))
                    cumulative_dead = dead_df[dead_df.iloc[:, 2] <= age].iloc[:, 4].sum() + dead_df[dead_df.iloc[:, 2] <= age].iloc[:, 5].sum()
                    
                    live_chickens = 54000 - cumulative_dead
                    total_live_weight = live_chickens * avg_weight / 1000
                    
                    if total_live_weight > 0:
                        fcr_value = cumulative_feed / total_live_weight
                        fcr_data[f'鸡舍{house}'] = fcr_value
                        all_fcr_values.append(fcr_value)
                except:
                    pass
            
            # 确定颜色标识的阈值
            min_fcr = min(all_fcr_values) if all_fcr_values else 0
            max_fcr = max(all_fcr_values) if all_fcr_values else 0
            
            # 第二轮：生成表格数据
            for house in sorted(age_data['鸡舍编号'].unique()):
                house_data = age_data[age_data['鸡舍编号'] == house]
                valid_data = house_data['均重(g)'].dropna()
                if len(valid_data) == 0: continue
                
                avg_weight = int(valid_data.mean())
                pos_info, layer_info = "", ""

                # 前后差异分析
                if '鸡笼编号' in house_data.columns and '层数' in house_data.columns:
                    max_pos_diff = 0
                    max_pos_info = ""
                    
                    for i, row1 in house_data.iterrows():
                        for j, row2 in house_data.iterrows():
                            if i != j and row1['鸡笼编号'] != row2['鸡笼编号']:
                                if pd.notna(row1['均重(g)']) and pd.notna(row2['均重(g)']):
                                    diff = abs(row1['均重(g)'] - row2['均重(g)'])
                                    if diff > max_pos_diff:
                                        max_pos_diff = diff
                                        if row1['均重(g)'] < row2['均重(g)']:
                                            min_row, max_row = row1, row2
                                        else:
                                            min_row, max_row = row2, row1
                                        
                                        min_pos = position_map.get(min_row['鸡笼编号'], f"{min_row['鸡笼编号']}")
                                        max_pos = position_map.get(max_row['鸡笼编号'], f"{max_row['鸡笼编号']}")
                                        max_pos_info = f"{min_pos}{min_row['层数']}{min_row['均重(g)']:.0f}g→{max_pos}{max_row['层数']}{max_row['均重(g)']:.0f}g"
                    
                    if max_pos_diff > 0:
                        pos_info = f"{max_pos_diff:.0f}g({max_pos_info})"
                
                # 上下差异分析
                if '鸡笼编号' in house_data.columns and '层数' in house_data.columns:
                    max_layer_diff = 0
                    max_layer_info = ""
                    
                    for cage in house_data['鸡笼编号'].unique():
                        cage_data = house_data[house_data['鸡笼编号'] == cage]
                        layer_avg = cage_data.groupby('层数')['均重(g)'].mean().dropna().round(0).astype(int)
                        if len(layer_avg) > 1:
                            cage_diff = layer_avg.max() - layer_avg.min()
                            if cage_diff > max_layer_diff:
                                max_layer_diff = cage_diff
                                cage_pos = position_map.get(cage, f"{cage}")
                                max_layer_info = f"{cage_pos}{layer_avg.idxmin()}{layer_avg.min()}g→{cage_pos}{layer_avg.idxmax()}{layer_avg.max()}g"
                    
                    if max_layer_diff > 0:
                        layer_info = f"{max_layer_diff}g({max_layer_info})"

                # 周增重和日增重
                weekly_gain = ""
                daily_gain = ""
                
                all_house_data = weight_df[weight_df['鸡舍编号'] == house].sort_values('日龄')
                current_index = all_house_data[all_house_data['日龄'] == age].index[0]
                
                prev_ages = all_house_data[all_house_data['日龄'] < age]['日龄']
                if len(prev_ages) > 0:
                    prev_age = prev_ages.max()
                    prev_weight_data = all_house_data[all_house_data['日龄'] == prev_age]
                    
                    if len(prev_weight_data) > 0:
                        prev_avg_weight = prev_weight_data['均重(g)'].mean()
                        current_avg_weight = avg_weight
                        
                        weight_gain = current_avg_weight - prev_avg_weight
                        weekly_gain = f"{weight_gain:.0f}g"
                        
                        days_diff = age - prev_age
                        if days_diff > 0:
                            daily_gain_value = weight_gain / days_diff
                            daily_gain = f"{daily_gain_value:.1f}g/天"
                
                # 料肉比颜色标识
                fcr = ""
                try:
                    feed_df = pd.read_excel(file_path, sheet_name=str(house))
                    cumulative_feed = feed_df[feed_df.iloc[:, 2] <= age].iloc[:, 3].sum()
                    
                    dead_df = pd.read_excel(file_path, sheet_name=str(house))
                    cumulative_dead = dead_df[dead_df.iloc[:, 2] <= age].iloc[:, 4].sum() + dead_df[dead_df.iloc[:, 2] <= age].iloc[:, 5].sum()
                    
                    live_chickens = 54000 - cumulative_dead
                    total_live_weight = live_chickens * avg_weight / 1000
                    
                    if total_live_weight > 0:
                        fcr_value = cumulative_feed / total_live_weight
                        
                        # 颜色标识：蓝色最低，红色最高
                        if all_fcr_values and len(all_fcr_values) > 1:
                            if abs(fcr_value - min_fcr) < 0.001:  # 避免浮点数精度问题
                                fcr = f"🔵{fcr_value:.2f}"  # 蓝色标识最低（最好）
                            elif abs(fcr_value - max_fcr) < 0.001:  # 避免浮点数精度问题
                                fcr = f"🔴{fcr_value:.2f}"  # 红色标识最高（最差）
                            else:
                                fcr = f"{fcr_value:.2f}"   # 普通显示中间值
                        else:
                            fcr = f"{fcr_value:.2f}"
                            
                except Exception as e:
                    fcr = "计算错误"
                
                table_data.append([f'鸡舍{house}', f'{avg_weight}g', pos_info, layer_info, weekly_gain, daily_gain, fcr])
            
            df_display = pd.DataFrame(table_data, columns=['鸡舍', '平均体重', '前后差异', '上下差异', '周增重', '日增重', '料肉比'])
            # 隐藏索引
            st.table(df_display.set_index('鸡舍'))

    if st.sidebar.button("开始分析"):
        with st.spinner("分析中..."):
            analyze_weight()

# 饲料预测
elif page == "饲料预测":
    st.title("🌾 精准饲料预测")
    
    def feed_forecast(days=5):
        std_df = pd.read_excel(file_path, sheet_name='采食标准')
        purchase_df = pd.read_excel(file_path, sheet_name='采购饲料记录')
        
        today = datetime.now()
        start_date = today
        end_date = today + timedelta(days=days-1)
        date_range = f"{start_date.month}月{start_date.day}号-{end_date.month}月{end_date.day}号"
        
        st.subheader(f"未来{days}天饲料需求({date_range})")
        st.write("---")
        
        total_demand = {}
        table_data = []
        total_gap = 0
        gap_details = []  # 存储各鸡舍缺料明细
        
        for house in range(1, 17):
            try:
                df = pd.read_excel(file_path, sheet_name=str(house))
                age = df.iloc[-1, 2]
                chickens = df.iloc[-1, 6]
                used = df.iloc[:, 3].sum()
                
                purchased = purchase_df[purchase_df['鸡舍编号'] == house]['采购饲料(kg)'].sum()
                stock = max(0, purchased - used) / 1000
                
                demand = {}
                for day in range(days):
                    future_age = age + day
                    if future_age > std_df['日龄'].max():
                        feed_info = std_df.iloc[-1]
                    else:
                        feed_info = std_df[std_df['日龄'] == future_age]
                        if feed_info.empty:
                            feed_info = std_df[std_df['日龄'] <= future_age].iloc[-1]
                        else:
                            feed_info = feed_info.iloc[0]
                    
                    feed_type = str(feed_info['料号'])
                    feed_amount = feed_info.iloc[1]
                    day_demand = (feed_amount * chickens) / 1000000
                    demand[feed_type] = demand.get(feed_type, 0) + day_demand
                
                total_needed = sum(demand.values())
                feed_types = "+".join(demand.keys())
                gap = max(0, total_needed - stock)
                total_gap += gap
                
                # 记录缺料的鸡舍信息
                if gap > 0:
                    # 对总缺料量进1取整
                    rounded_gap = math.ceil(gap)
                    # 计算每种料号的具体需求量（进1取整）
                    feed_demand_details = []
                    for feed_type, feed_demand in demand.items():
                        rounded_demand = math.ceil(feed_demand)  # 进1取整
                        feed_demand_details.append(f"{feed_type}({rounded_demand:.0f}吨)")
                    demand_detail = "+".join(feed_demand_details)
                    # 显示库存量（进1取整）
                    rounded_stock = math.ceil(stock)
                    gap_details.append([f'鸡舍{house}', f'{rounded_stock:.0f}吨', f'{rounded_gap:.0f}吨', demand_detail])
                
                if total_needed <= stock:
                    status = f"🟢充足({feed_types})"
                    amount = f"{stock:.1f}吨"
                else:
                    status = f"🔴缺料({feed_types})"
                    amount = f"{stock:.1f}吨"
                
                # 表格中的缺料量也进1取整
                rounded_gap_display = math.ceil(gap) if gap > 0 else 0
                table_data.append([f'鸡舍{house}', f'{age}日龄', f'{chickens}只', status, amount, f'{rounded_gap_display:.0f}吨'])
                
                for feed_type, feed_demand in demand.items():
                    total_demand[feed_type] = total_demand.get(feed_type, 0) + feed_demand
                
            except Exception as e:
                # 添加异常信息显示，便于调试
                table_data.append([f'鸡舍{house}', '数据异常', '-', '❌错误', '-', '-'])
        
        df_table = pd.DataFrame(table_data, 
            columns=['鸡舍', '当前日龄', '存栏数', '状态', '库存', '缺料量'])
        # 隐藏索引
        st.table(df_table.set_index('鸡舍'))
        
        st.write("---")
        st.subheader("📊 需求汇总")
        
        # 显示各料号总需求（进1取整）
        summary_data = []
        total_demand_rounded = 0
        for feed_type, amount in sorted(total_demand.items()):
            rounded_amount = math.ceil(amount)  # 进1取整
            total_demand_rounded += rounded_amount
            summary_data.append([f'{feed_type}料号', f'{rounded_amount:.0f}吨'])
        
        # 总缺料量进1取整
        total_gap_rounded = math.ceil(total_gap)
        summary_data.append(['**总计**', f'**{total_demand_rounded:.0f}吨**'])
        summary_data.append(['**总缺料量**', f'**{total_gap_rounded:.0f}吨**'])
        
        df_summary = pd.DataFrame(summary_data, columns=['料号', '总需求'])
        # 隐藏索引
        st.table(df_summary.set_index('料号'))
        
        # 显示各鸡舍缺料明细
        if gap_details:
            st.subheader(f"🚨 缺料鸡舍明细({date_range})")
            gap_df = pd.DataFrame(gap_details, columns=['鸡舍', '库存量', '缺料量', '未来料量合计'])
            # 隐藏索引
            st.table(gap_df.set_index('鸡舍'))
        else:
            st.success("✅ 所有鸡舍饲料充足，无需补料")
    
    days = st.sidebar.slider("预测天数", 1, 7, 3)
    if st.sidebar.button("开始预测"):
        with st.spinner("预测中..."):
            feed_forecast(days=days)

# 进料量分析
elif page == "进料量分析":
    st.title("📦 各栋进料量统计")
    
    def analyze_feed_intake():
        purchase_df = pd.read_excel(file_path, sheet_name='采购饲料记录')
        
        # 将料号统一转换为字符串类型
        purchase_df['料号'] = purchase_df['料号'].astype(str)
        
        # 获取所有料号并排序
        feed_types = sorted(purchase_df['料号'].unique())
        
        # 创建横纵排表格
        table_data = []
        for house in range(1, 17):
            row = [f'鸡舍{house}']
            house_data = purchase_df[purchase_df['鸡舍编号'] == house]
            
            # 各料号进料量
            total_house = 0
            for feed_type in feed_types:
                feed_amount = house_data[house_data['料号'] == feed_type]['采购饲料(kg)'].sum() / 1000
                row.append(f"{feed_amount:.1f}" if feed_amount > 0 else "-")
                total_house += feed_amount
            
            # 累计量
            row.append(f"{total_house:.1f}")
            table_data.append(row)
        
        # 添加最后一行累计量
        total_row = ['累计']
        grand_total = 0
        
        for i, feed_type in enumerate(feed_types):
            feed_total = 0
            for house_data in table_data:
                value = house_data[i + 1]  # i+1 因为第一列是鸡舍名称
                if value != '-':
                    feed_total += float(value)
            total_row.append(f"{feed_total:.1f}")
            grand_total += feed_total
        
        total_row.append(f"{grand_total:.1f}")
        table_data.append(total_row)
        
        # 表头
        headers = ['鸡舍'] + feed_types + ['累计(吨)']
        
        # 显示表格
        df_display = pd.DataFrame(table_data, columns=headers)
        # 隐藏索引
        st.table(df_display.set_index('鸡舍'))
    
    if st.sidebar.button("开始分析"):
        with st.spinner("统计中..."):
            analyze_feed_intake()

# 每日分析
elif page == "每日分析":
    st.title("📊 每日采食分析")
    
    def analyze_daily_feed():
        # 读取采食标准表
        std_df = pd.read_excel(file_path, sheet_name='采食标准')
        
        # 手动输入日期
        col1, col2, col3 = st.columns(3)
        with col1:
            year = st.number_input("年份", min_value=2023, max_value=2030, value=2024)
        with col2:
            month = st.number_input("月份", min_value=1, max_value=12, value=datetime.now().month)
        with col3:
            day = st.number_input("日期", min_value=1, max_value=31, value=datetime.now().day)
        
        # 构造日期对象
        selected_date = datetime(year, month, day).date()
        
        table_data = []
        # 循环分析每个鸡舍
        for house in range(1, 17):
            try:
                # 读取鸡舍数据
                df = pd.read_excel(file_path, sheet_name=str(house))
                
                # 假设第一列是日期，转换为日期格式
                df['日期'] = pd.to_datetime(df.iloc[:, 0]).dt.date
                
                # 查找匹配日期的数据
                matching_rows = df[df['日期'] == selected_date]
                if matching_rows.empty:
                    continue
                
                # 取第一行匹配的数据
                row_data = matching_rows.iloc[0]
                current_age = row_data.iloc[2]  # 日龄
                daily_feed = row_data.iloc[3]   # 当日采食量(kg)
                current_chickens = row_data.iloc[6]  # 存栏量
                
                # 计算累计采食量（从开始到选中日期）
                cumulative_feed = df[df['日期'] <= selected_date].iloc[:, 3].sum()
                
                # 计算累计死淘数（从开始到选中日期）
                cumulative_dead = df[df['日期'] <= selected_date].iloc[:, 4].sum() + df[df['日期'] <= selected_date].iloc[:, 5].sum()
                
                # 计算成活率（假设初始54000只）
                initial_chickens = 54000
                survival_rate = ((initial_chickens - cumulative_dead) / initial_chickens * 100).round(1)
                
                # 获取标准数据
                std_data = std_df[std_df['日龄'] == current_age]
                if not std_data.empty:
                    # 修正：当日标准采食量应该是该日龄的单日采食标准
                    # 如果标准表中有"单日采食量"列，使用该列；否则使用累计采食量的差值
                    if '单日采食量/克' in std_df.columns:
                        # 如果有单日采食量列，直接使用
                        std_daily_per_chicken = std_data.iloc[0]['单日采食量/克']
                    else:
                        # 如果没有单日采食量列，计算当日标准 = 当日累计标准 - 前一日累计标准
                        current_std = std_data.iloc[0]['单只鸡累计采食量/克']
                        # 找前一日标准
                        prev_std_data = std_df[std_df['日龄'] == current_age - 1]
                        if not prev_std_data.empty:
                            prev_std = prev_std_data.iloc[0]['单只鸡累计采食量/克']
                            std_daily_per_chicken = current_std - prev_std
                        else:
                            # 如果是第一天，直接用累计值
                            std_daily_per_chicken = current_std
                    
                    # 计算总标准采食量 = 单只标准 × 存栏量 ÷ 1000 (kg)
                    std_daily_total = (std_daily_per_chicken * current_chickens) / 1000
                    
                    # 计算当日差异
                    daily_diff = daily_feed - std_daily_total
                    # 设置差异状态
                    if daily_diff > 200:
                        diff_status = f"多{daily_diff:.0f}kg"
                    elif daily_diff < -200:
                        diff_status = f"少{-daily_diff:.0f}kg"
                    else:
                        diff_status = "正常"
                    
                    # 计算累计标准 = 当前日龄的累计标准 × 存栏量 ÷ 1000
                    std_cumulative_total = (std_data.iloc[0]['单只鸡累计采食量/克'] * current_chickens) / 1000
                    # 计算累计差异
                    cumulative_diff = cumulative_feed - std_cumulative_total
                    # 设置累计状态
                    if cumulative_diff > 200:
                        cumulative_status = f"多{cumulative_diff:.0f}kg"
                    elif cumulative_diff < -200:
                        cumulative_status = f"少{-cumulative_diff:.0f}kg"
                    else:
                        cumulative_status = "正常"
                    
                    # 添加到表格数据
                    table_data.append([
                        f'鸡舍{house}',
                        f'{current_age}日龄',
                        f'{daily_feed:.0f}kg',
                        f'{std_daily_total:.0f}kg',
                        diff_status,
                        cumulative_status,
                        f'{cumulative_dead:.0f}只',
                        f'{survival_rate}%'
                    ])
                    
            except Exception as e:
                # 显示错误信息以便调试
                st.error(f"鸡舍{house}分析错误: {str(e)}")
                continue
        
        # 显示结果表格
        if table_data:
            st.subheader(f"{selected_date} 每日分析结果")
            df_display = pd.DataFrame(table_data, 
                columns=['鸡舍', '日龄', '当日采食', '标准采食', '差异', '累计差异', '累计死淘', '成活率'])
            # 隐藏索引
            st.table(df_display.set_index('鸡舍'))
            
            # 显示计算说明
            with st.expander("计算说明"):
                st.write("""
                - **当日采食**: 实际记录的当日采食量(kg)
                - **标准采食**: 标准单日采食量(克/只) × 存栏量 ÷ 1000
                - **差异**: 实际采食量与标准采食量的差值
                - **累计差异**: 实际累计采食量与标准累计采食量的差值
                - **累计死淘**: 从开始到当前日期的累计死淘数量
                - **成活率**: (初始54000只 - 累计死淘) ÷ 54000 × 100%
                """)
        else:
            st.warning(f"{selected_date} 无数据")
    
    # 直接运行分析
    analyze_daily_feed()
