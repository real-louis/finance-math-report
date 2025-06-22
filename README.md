# finance-math-report

import numpy as np
import warnings

# --- 參數設定 ---

# 1. 個人與情境設定
current_age = 25
retire_age = 50
simulation_end_age =80 # 模擬計算至100歲，評估長壽風險

# 2. 薪資與工作參數
salary_start = 100000  # 起始月薪
employer_contrib_rate = 0.06 # 雇主提撥率 (法定>=6%)
self_contrib_rate = 0.06     # 個人自願提撥率 (法定<=6%)
insurance_years_at_retire = retire_age - current_age # 預計總勞保年資

# 3. 個人儲蓄與開銷
current_savings = 100000
monthly_saving = 10000
monthly_expense_healthy_today = 40000 # 以"現值"計算的退休後每月開銷
monthly_expense_unhealthy_today = 60000 # 以"現值"計算的退休後非健康期開銷
healthy_years_after_retire = 37 # 退休後健康狀態的年數

# 4. 蒙地卡羅模擬設定
num_simulations = 10000 # 模擬次數
# 平均年化報酬率 (預期值)
salary_growth_mean = 0.2
inflation_mean = 0.02
gov_pension_return_mean = 0.04 # 勞退基金(政府管理)長期平均報酬
personal_saving_return_mean = 0.1 # 個人投資組合長期平均報酬
# 年化波動率 (標準差)
salary_growth_std = 0.05
inflation_std = 0.015
gov_pension_return_std = 0.08
personal_saving_return_std = 0.12

# 忽略Numpy在計算過程可能產生的浮點數溢位警告
warnings.filterwarnings('ignore', r'overflow encountered in double_scalars')
warnings.filterwarnings('ignore', r'invalid value encountered in double_scalars')

# --- 核心邏輯 ---

def get_pension_contribution_level(monthly_salary):
    """根據月薪，查詢勞工退休金月提繳分級表，返回對應的提繳工資。"""
    # 簡化版的2024年勞工退休金月提繳分級表
    if monthly_salary <= 1500: return 1500
    if monthly_salary <= 3000: return 3000
    if monthly_salary <= 4500: return 4500
    if monthly_salary <= 6000: return 6000
    if monthly_salary <= 7500: return 7500
    if monthly_salary <= 8700: return 8700
    if monthly_salary <= 9900: return 9900
    if monthly_salary <= 11100: return 11100
    if monthly_salary <= 12540: return 12540
    if monthly_salary <= 13500: return 13500
    if monthly_salary <= 15840: return 15840
    if monthly_salary <= 16500: return 16500
    if monthly_salary <= 17280: return 17280
    if monthly_salary <= 19047: return 19047
    if monthly_salary <= 20008: return 20008
    if monthly_salary <= 21009: return 21009
    if monthly_salary <= 22000: return 22000
    if monthly_salary <= 23100: return 23100
    if monthly_salary <= 24000: return 24000
    if monthly_salary <= 25250: return 25250
    if monthly_salary <= 26400: return 26400
    if monthly_salary <= 27600: return 27600
    if monthly_salary <= 28800: return 28800
    if monthly_salary <= 30300: return 30300
    if monthly_salary <= 31800: return 31800
    if monthly_salary <= 33300: return 33300
    if monthly_salary <= 34800: return 34800
    if monthly_salary <= 36300: return 36300
    if monthly_salary <= 38200: return 38200
    if monthly_salary <= 40100: return 40100
    if monthly_salary <= 42000: return 42000
    if monthly_salary <= 43900: return 43900
    if monthly_salary >= 150000: return 150000 # 級距上限
    # 45800元是勞保天花板，但勞退提繳級距更高
    return 45800

def run_single_simulation():
    """運行一次完整的生命週期模擬"""
    
    # --- Part 1: 資產累積期 ---
    years_worked = retire_age - current_age
    salary = salary_start
    savings = current_savings
    gov_pension_fund = 0 # 勞退雇主提撥部分
    
    # 記錄所有工作年份的月投保薪資，用於計算勞保年金
    insurance_salary_history = []

    for _ in range(years_worked):
        # 薪資成長 (隨機)
        salary_growth = np.random.normal(salary_growth_mean, salary_growth_std)
        salary *= (1 + salary_growth)
        
        # 勞退提撥 (依據提繳級距表)
        contribution_base = get_pension_contribution_level(salary)
        # 勞保投保薪資有其上限(45800)，但此處以提繳薪資為準以簡化
        insurance_salary_history.append(contribution_base)

        # 勞退基金成長 (隨機)
        gov_pension_return = np.random.normal(gov_pension_return_mean, gov_pension_return_std)
        gov_pension_fund = gov_pension_fund * (1 + gov_pension_return) + \
                           (contribution_base * employer_contrib_rate * 12) + \
                           (contribution_base * self_contrib_rate * 12)

        # 個人儲蓄成長 (隨機)
        saving_return = np.random.normal(personal_saving_return_mean, personal_saving_return_std)
        savings = savings * (1 + saving_return) + (monthly_saving * 12)

    # --- Part 2: 退休金計算 ---
    
    # 1. 計算勞保年金 (最高60個月平均薪資，二擇優)
    insurance_salary_history.sort(reverse=True)
    highest_60_months_avg = sum(insurance_salary_history[:60]) / 60
    
    pension_formula_1 = highest_60_months_avg * insurance_years_at_retire * 0.00775 + 3000
    pension_formula_2 = highest_60_months_avg * insurance_years_at_retire * 0.0155
    insurance_pension_monthly = max(pension_formula_1, pension_formula_2)

    # 退休時總資產
    total_pension_at_retire = gov_pension_fund
    total_savings_at_retire = savings

    # --- Part 3: 退休提領期 ---
    
    pension_balance = total_pension_at_retire
    saving_balance = total_savings_at_retire
    
    years_after_retire = simulation_end_age - retire_age
    
    for i in range(years_after_retire):
        # 隨機通膨
        inflation = np.random.normal(inflation_mean, inflation_std)
        
        # 計算當年開銷 (受通膨影響)
        if i < healthy_years_after_retire:
            expense = (monthly_expense_healthy_today * 12) * ((1 + inflation) ** i)
        else:
            expense = (monthly_expense_unhealthy_today * 12) * ((1 + inflation) ** i)
            
        # 勞保年金 (假設跟隨長期通膨調整，此處簡化)
        # 法規為CPI累計>5%才調，此處簡化為每年調整
        insurance_pension_annual = insurance_pension_monthly * 12
        insurance_pension_monthly *= (1 + inflation)

        # 勞退金提領 (分年領取)
        remaining_years = years_after_retire - i
        pension_withdraw = pension_balance / remaining_years if remaining_years > 0 and pension_balance > 0 else 0
        pension_balance -= pension_withdraw
        
        # 總收入與結餘
        total_income = insurance_pension_annual + pension_withdraw
        net_income = total_income - expense
        
        # 剩餘存款進行再投資
        saving_return = np.random.normal(personal_saving_return_mean, personal_saving_return_std)
        saving_balance = (saving_balance + net_income) * (1 + saving_return)

        if saving_balance < 0:
            saving_balance = 0 # 錢花完了
            break
            
    return saving_balance

# --- 主程式：運行蒙地卡羅模擬 ---
final_balances = []
print(f"正在運行 {num_simulations} 次蒙地卡羅模擬...")

for i in range(num_simulations):
    final_balances.append(run_single_simulation())
    if (i + 1) % (num_simulations / 10) == 0:
        print(f"  ...已完成 {i + 1} / {num_simulations} 次模擬")

final_balances = np.array(final_balances)

# --- 結果分析與輸出 ---
success_rate = np.sum(final_balances > 0) / num_simulations
median_balance = np.median(final_balances)
percentile_10 = np.percentile(final_balances, 10)
percentile_90 = np.percentile(final_balances, 90)

print("\n--- 退休財務規劃模擬結果 ---")
print(f"模擬情境：從 {current_age} 歲工作至 {retire_age} 歲，評估至 {simulation_end_age} 歲的財務狀況。")
print(f"模擬次數：{num_simulations} 次\n")

print(f"退休規劃成功率: {success_rate:.1%}")
print(f"  (在 {simulation_end_age} 歲時，退休金仍有餘額的機率)\n")

print("最終資產分佈情況：")
print(f"  悲觀情境 (10百分位): NT$ {percentile_10:,.0f}")
print(f"  中位數情境 (50百分位): NT$ {median_balance:,.0f}")
print(f"  樂觀情境 (90百分位): NT$ {percentile_90:,.0f}\n")

if success_rate < 0.8:
    print("提醒：成功率低於80%，建議考慮以下方式提高成功率：")
    print("1. 提高每月儲蓄金額或個人自提比例。")
    print("2. 延後退休年齡。")
    print("3. 調整個人投資組合以期獲得更高的長期回報（同時也可能承受更高風險）。")
    print("4. 重新評估退休後的花費。")
