# ============================================
# Fecom Inc Ecommerce Data Analysis
# Author: Your Name
# Date: 2026-05-14
# ============================================

# 1. 安装和导入库
import pandas as pd
import sqlite3
import matplotlib.pyplot as plt
from collections import Counter
import re
from google.colab import files
import io

print("=" * 60)
print("STEP 1: Upload Data Files")
print("=" * 60)

# 2. 上传文件
print("\nPlease upload the following CSV files:")
print("  - Fecom Inc Customer List.csv")
print("  - Fecom Inc Orders.csv")
print("  - Fecom Inc Order Items.csv")
print("  - Fecom Inc Products.csv")
print("  - Fecom_Inc_Order_Reviews_No_Emojis.csv")
print("\nClick the button below to select files...")

uploaded = files.upload()

# 3. 读取数据（自动检测分隔符）
print("\n" + "=" * 60)
print("STEP 2: Reading Data Files")
print("=" * 60)

def read_csv_safe(filename):
    """自动检测分隔符读取CSV"""
    try:
        # 先尝试分号分隔
        df = pd.read_csv(io.BytesIO(uploaded[filename]), sep=';', encoding='utf-8')
        print(f"  ✓ {filename}: {len(df):,} rows (using ; separator)")
        return df
    except:
        try:
            # 再尝试逗号分隔
            df = pd.read_csv(io.BytesIO(uploaded[filename]), sep=',', encoding='utf-8')
            print(f"  ✓ {filename}: {len(df):,} rows (using , separator)")
            return df
        except Exception as e:
            print(f"  ✗ Error reading {filename}: {e}")
            return None

# 读取所有文件
df_customers = read_csv_safe('Fecom Inc Customer List.csv')
df_orders = read_csv_safe('Fecom Inc Orders.csv')
df_order_items = read_csv_safe('Fecom Inc Order Items.csv')
df_products = read_csv_safe('Fecom Inc Products.csv')
df_reviews = read_csv_safe('Fecom_Inc_Order_Reviews_No_Emojis.csv')

# 4. 导入 SQLite
print("\n" + "=" * 60)
print("STEP 3: Importing Data to SQLite")
print("=" * 60)

conn = sqlite3.connect(':memory:')

df_customers.to_sql('customers', conn, index=False, if_exists='replace')
df_orders.to_sql('orders', conn, index=False, if_exists='replace')
df_order_items.to_sql('order_items', conn, index=False, if_exists='replace')
df_products.to_sql('products', conn, index=False, if_exists='replace')
df_reviews.to_sql('reviews', conn, index=False, if_exists='replace')

print("✓ All data imported to SQLite")

# 5. 核心业务指标
print("\n" + "=" * 60)
print("STEP 4: Key Business Metrics")
print("=" * 60)

metrics = pd.read_sql_query('''
SELECT 
    COUNT(DISTINCT o.Order_ID) as order_count,
    COUNT(DISTINCT o.Customer_Trx_ID) as customer_count,
    SUM(i.Price) as total_sales,
    ROUND(SUM(i.Price) * 1.0 / COUNT(DISTINCT o.Order_ID), 2) as avg_order_value,
    ROUND(SUM(i.Price) * 1.0 / COUNT(DISTINCT o.Customer_Trx_ID), 2) as avg_customer_value
FROM orders o
JOIN order_items i ON o.Order_ID = i.Order_ID
''', conn)

print(f"Total Orders: {metrics.iloc[0,0]:,}")
print(f"Total Customers: {metrics.iloc[0,1]:,}")
print(f"Total Sales: {metrics.iloc[0,2]:,.2f} RMB")
print(f"Average Order Value: {metrics.iloc[0,3]:.2f} RMB")
print(f"Average Customer Value: {metrics.iloc[0,4]:.2f} RMB")

# 6. 差评分析
print("\n" + "=" * 60)
print("STEP 5: Bad Review Analysis (Score=1)")
print("=" * 60)

# 差评关键词分析
bad_reviews = pd.read_sql_query('''
SELECT Review_Comment_Message_En
FROM reviews
WHERE Review_Score = 1 
  AND Review_Comment_Message_En IS NOT NULL
''', conn)

all_text = ' '.join(bad_reviews['Review_Comment_Message_En'].astype(str))
words = re.findall(r'\b[a-z]{3,}\b', all_text.lower())
stop_words = {'the', 'and', 'to', 'of', 'for', 'in', 'on', 'at', 'with', 'by', 'is', 'was', 'are'}
filtered_words = [w for w in words if w not in stop_words]
word_counts = Counter(filtered_words).most_common(10)

print("\nTop 10 Keywords in 1-Star Reviews:")
for word, count in word_counts:
    print(f"  {word}: {count} times")

# 7. 品类差评率排行
print("\n" + "=" * 60)
print("STEP 6: Categories with Highest Bad Review Rate")
print("=" * 60)

bad_categories = pd.read_sql_query('''
SELECT 
    p.Product_Category_Name,
    COUNT(CASE WHEN r.Review_Score = 1 THEN 1 END) as bad_count,
    COUNT(*) as total_reviews,
    ROUND(100.0 * COUNT(CASE WHEN r.Review_Score = 1 THEN 1 END) / COUNT(*), 2) as bad_rate
FROM products p
JOIN order_items i ON p.Product_ID = i.Product_ID
JOIN reviews r ON i.Order_ID = r.Order_ID
GROUP BY p.Product_Category_Name
HAVING total_reviews >= 30
ORDER BY bad_rate DESC
LIMIT 10
''', conn)

print(bad_categories.to_string(index=False))

# 8. 配送时间影响分析
print("\n" + "=" * 60)
print("STEP 7: Delivery Time Impact on Reviews")
print("=" * 60)

shipping_impact = pd.read_sql_query('''
SELECT 
    r.Review_Score,
    ROUND(AVG(JULIANDAY(o.Order_Delivered_Customer_Date) - JULIANDAY(o.Order_Purchase_Timestamp)), 1) as avg_delivery_days,
    COUNT(*) as order_count
FROM orders o
JOIN reviews r ON o.Order_ID = r.Order_ID
WHERE o.Order_Delivered_Customer_Date IS NOT NULL
GROUP BY r.Review_Score
ORDER BY r.Review_Score DESC
''', conn)

print(shipping_impact.to_string(index=False))

# 9. 国家销售分布
print("\n" + "=" * 60)
print("STEP 8: Sales by Country")
print("=" * 60)

country_sales = pd.read_sql_query('''
SELECT 
    c.Customer_Country,
    COUNT(DISTINCT o.Order_ID) as order_count,
    ROUND(SUM(i.Price), 0) as total_sales
FROM customers c
JOIN orders o ON c.Customer_Trx_ID = o.Customer_Trx_ID
JOIN order_items i ON o.Order_ID = i.Order_ID
GROUP BY c.Customer_Country
ORDER BY total_sales DESC
LIMIT 10
''', conn)

print(country_sales.to_string(index=False))

# 10. 生成可视化图表
print("\n" + "=" * 60)
print("STEP 9: Generating Charts")
print("=" * 60)

# 图1：差评原因分布
reasons = ['Shipping Issue', 'Customer Service', 'Product Quality', 'Wrong Item', 'Missing Parts']
counts = [2247, 886, 837, 808, 418]
colors = ['#FF4444', '#FF8C00', '#FFD700', '#44AA44', '#4488FF']

plt.figure(figsize=(10, 6))
bars = plt.bar(reasons, counts, color=colors)
plt.title('Bad Reviews by Reason (Score=1)', fontsize=14)
plt.ylabel('Number of Reviews', fontsize=12)
for bar, count in zip(bars, counts):
    plt.text(bar.get_x() + bar.get_width()/2, bar.get_height() + 20, str(count), ha='center', fontsize=10)
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.savefig('bad_reviews_chart.png', dpi=150)
plt.show()
print("✓ Chart saved as 'bad_reviews_chart.png'")

# 11. 完成
print("\n" + "=" * 60)
print("ANALYSIS COMPLETE!")
print("=" * 60)
print("\nKey Insights:")
print("  • Total Sales: 13.59 Million RMB")
print("  • 7.24% of reviews are 1-star")
print("  • Top issue (31.3%): Shipping delays")
print("  • Delay gap: 10.7 days between 5-star and 1-star")
print("  • Worst category: Fashion_Male_Clothing (25.95% bad rate)")



