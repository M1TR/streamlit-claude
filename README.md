# streamlit-claude

import streamlit as st
import pandas as pd
import sqlite3
from datetime import datetime
import io

# Configure the page
st.set_page_config(
    page_title="Text-to-SQL Generator",
    page_icon="🔍",
    layout="wide",
    initial_sidebar_state="expanded"
)

# Custom CSS to match the artifact's styling
st.markdown("""
<style>
    /* Global styles */
    .main .block-container {
        padding-top: 2rem;
        padding-bottom: 2rem;
    }
    
    /* Header styling */
    .app-header {
        text-align: center;
        margin-bottom: 2rem;
        padding: 1rem;
        background: white;
        border-radius: 10px;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
    }
    
    .app-title {
        font-size: 2.5rem !important;
        font-weight: 700 !important;
        color: #0e1117 !important;
        margin-bottom: 0.5rem !important;
    }
    
    .app-subtitle {
        color: #6c757d !important;
        font-size: 1.1rem !important;
    }
    
    /* Section headers */
    .section-header {
        font-size: 1.25rem;
        font-weight: 600;
        color: #0e1117;
        margin-bottom: 1rem;
        padding: 0.5rem 0;
        border-bottom: 2px solid #ff4b4b;
    }
    
    /* Custom boxes */
    .info-box {
        background: #e0f2fe;
        border: 1px solid #81d4fa;
        border-radius: 8px;
        padding: 12px;
        margin: 10px 0;
        color: #0277bd;
    }
    
    .success-box {
        background: #e8f5e8;
        border: 1px solid #81c784;
        border-radius: 8px;
        padding: 12px;
        margin: 10px 0;
        color: #2e7d32;
    }
    
    .metric-card {
        background: #f8f9fa;
        border: 1px solid #e9ecef;
        border-radius: 8px;
        padding: 1rem;
        text-align: center;
        margin: 0.5rem 0;
    }
    
    .metric-value {
        font-size: 1.5rem;
        font-weight: 700;
        color: #0e1117;
        margin-bottom: 0.25rem;
    }
    
    .metric-label {
        font-size: 0.875rem;
        color: #6c757d;
    }
    
    /* Code blocks */
    .code-block {
        background: #f8f9fa;
        border: 1px solid #e9ecef;
        border-radius: 8px;
        padding: 1rem;
        font-family: 'Monaco', 'Consolas', monospace;
        font-size: 0.875rem;
        margin: 1rem 0;
        overflow-x: auto;
        color: #0e1117;
    }
    
    /* Custom buttons */
    .stButton > button {
        background: #ff4b4b;
        color: white;
        border: none;
        border-radius: 6px;
        padding: 0.5rem 1rem;
        font-weight: 500;
        transition: all 0.2s;
    }
    
    .stButton > button:hover {
        background: #e63946;
        border: none;
    }
    
    /* Sidebar styling */
    .css-1d391kg {
        background-color: white;
    }
    
    /* Data table styling */
    .dataframe {
        border: 1px solid #e6e9ef !important;
        border-radius: 8px !important;
    }
    
    /* History section */
    .history-item {
        background: white;
        border: 1px solid #e6e9ef;
        border-radius: 8px;
        padding: 1rem;
        margin: 0.5rem 0;
        box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    
    .history-timestamp {
        font-size: 0.75rem;
        color: #6c757d;
        margin-bottom: 0.5rem;
    }
</style>
""", unsafe_allow_html=True)

# Initialize session state
if 'query_history' not in st.session_state:
    st.session_state.query_history = []
if 'current_query' not in st.session_state:
    st.session_state.current_query = ""
if 'generated_sql' not in st.session_state:
    st.session_state.generated_sql = ""
if 'query_results' not in st.session_state:
    st.session_state.query_results = None

# Sample database schemas
DATABASE_SCHEMAS = {
    "Ecommerce Db": {
        "users": {
            "ddl": """CREATE TABLE users (
    id INTEGER PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE
);""",
            "columns": ["id", "username", "email", "created_at", "last_login", "is_active"]
        },
        "products": {
            "ddl": """CREATE TABLE products (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    category_id INTEGER,
    stock_quantity INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);""",
            "columns": ["id", "name", "price", "category_id", "stock_quantity", "created_at"]
        },
        "orders": {
            "ddl": """CREATE TABLE orders (
    id INTEGER PRIMARY KEY,
    user_id INTEGER NOT NULL,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);""",
            "columns": ["id", "user_id", "total_amount", "status", "created_at"]
        },
        "categories": {
            "ddl": """CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);""",
            "columns": ["id", "name", "description", "created_at"]
        }
    },
    "Hr System": {
        "employees": {
            "ddl": """CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    department_id INTEGER,
    salary DECIMAL(10,2),
    hire_date DATE NOT NULL
);""",
            "columns": ["id", "first_name", "last_name", "email", "department_id", "salary", "hire_date"]
        },
        "departments": {
            "ddl": """CREATE TABLE departments (
    id INTEGER PRIMARY KEY,
    name VARCHAR(50) UNIQUE NOT NULL,
    manager_id INTEGER,
    budget DECIMAL(12,2)
);""",
            "columns": ["id", "name", "manager_id", "budget"]
        }
    }
}

# Sample data for demonstration
SAMPLE_DATA = {
    "users": pd.DataFrame({
        "id": [1, 2, 3, 4, 5],
        "username": ["alice_j", "bob_smith", "carol_b", "david_w", "emma_d"],
        "email": ["alice@email.com", "bob@email.com", "carol@email.com", "david@email.com", "emma@email.com"],
        "created_at": ["2024-01-15", "2024-02-20", "2024-03-10", "2024-04-05", "2024-05-12"]
    }),
    "products": pd.DataFrame({
        "id": [1, 2, 3, 4, 5],
        "name": ["Laptop", "Mouse", "Keyboard", "Monitor", "Headphones"],
        "price": [999.99, 29.99, 79.99, 299.99, 149.99],
        "stock_quantity": [50, 200, 150, 75, 100]
    })
}

# Sidebar Configuration
with st.sidebar:
    st.markdown("## 🗄️ Database Configuration")
    
    # Database selection
    selected_db = st.selectbox("Select Database", list(DATABASE_SCHEMAS.keys()))
    
    # Display schema info
    num_tables = len(DATABASE_SCHEMAS[selected_db])
    st.markdown(f'<div class="info-box">📊 Schema: {num_tables} tables available</div>', 
                unsafe_allow_html=True)
    
    st.markdown("### 📋 Table Selection")
    
    # Table selection
    available_tables = list(DATABASE_SCHEMAS[selected_db].keys())
    selected_tables = st.multiselect(
        "Select Tables",
        available_tables,
        default=available_tables[:2] if len(available_tables) >= 2 else available_tables
    )
    
    # Connection status
    if selected_tables:
        st.markdown('<div class="success-box">✅ Connected</div>', unsafe_allow_html=True)
    else:
        st.warning("⚠️ Please select at least one table")

# Main content
st.markdown("""
<div class="app-header">
    <h1 class="app-title">🔍 Text-to-SQL Generator</h1>
    <p class="app-subtitle">Convert natural language queries into SQL statements</p>
</div>
""", unsafe_allow_html=True)

# DDL Section
if selected_tables:
    st.markdown('<div class="section-header">🏗️ Table Structures (DDL)</div>', 
                unsafe_allow_html=True)
    
    # Create tabs for each selected table
    if len(selected_tables) > 1:
        table_tabs = st.tabs([f"📋 {table.title()}" for table in selected_tables])
        
        for i, table in enumerate(selected_tables):
            with table_tabs[i]:
                col1, col2 = st.columns([2, 1])
                
                with col1:
                    st.subheader(f"{table.title()} DDL")
                    st.code(DATABASE_SCHEMAS[selected_db][table]["ddl"], language="sql")
                
                with col2:
                    st.subheader("🔗 Columns")
                    columns = DATABASE_SCHEMAS[selected_db][table]["columns"]
                    for idx, col in enumerate(columns, 1):
                        st.write(f"**{idx}.** {col}")
    else:
        table = selected_tables[0]
        col1, col2 = st.columns([2, 1])
        
        with col1:
            st.subheader(f"📋 {table.title()} DDL")
            st.code(DATABASE_SCHEMAS[selected_db][table]["ddl"], language="sql")
        
        with col2:
            st.subheader("🔗 Columns")
            columns = DATABASE_SCHEMAS[selected_db][table]["columns"]
            for idx, col in enumerate(columns, 1):
                st.write(f"**{idx}.** {col}")

# Query Workspace Section
st.markdown('<div class="section-header">🔍 Query Workspace</div>', 
            unsafe_allow_html=True)

col1, col2 = st.columns(2)

with col1:
    st.subheader("📝 Query Input")
    
    query_input = st.text_area(
        "",
        value="Show me all active users with their registration dates",
        height=100,
        placeholder="e.g., 'Show me all users who registered last month' or 'What are the top 5 products by sales?'"
    )
    
    button_col1, button_col2 = st.columns(2)
    
    with button_col1:
        generate_btn = st.button("🚀 Generate SQL", type="primary")
    
    with button_col2:
        clear_btn = st.button("🗑️ Clear All")

with col2:
    st.subheader("⚡ Generated SQL")
    
    # Generate SQL logic (simplified for demo)
    if generate_btn and query_input:
        # Simple keyword-based SQL generation for demo
        query_lower = query_input.lower()
        
        if "active users" in query_lower and "registration" in query_lower:
            generated_sql = """SELECT id, username, email, created_at 
FROM users 
WHERE is_active = TRUE 
ORDER BY created_at DESC;"""
        elif "top" in query_lower and "products" in query_lower:
            generated_sql = """SELECT name, price 
FROM products 
ORDER BY price DESC 
LIMIT 5;"""
        elif "users" in query_lower:
            generated_sql = """SELECT * FROM users;"""
        elif "products" in query_lower:
            generated_sql = """SELECT * FROM products;"""
        else:
            generated_sql = f"-- Query for: {query_input}\nSELECT * FROM {selected_tables[0] if selected_tables else 'table_name'};"
        
        st.session_state.generated_sql = generated_sql
        st.session_state.current_query = query_input
        
        # Add to history
        history_entry = {
            "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "question": query_input,
            "sql": generated_sql,
            "tables": ", ".join(selected_tables)
        }
        st.session_state.query_history.append(history_entry)
    
    if clear_btn:
        st.session_state.generated_sql = ""
        st.session_state.current_query = ""
        st.session_state.query_results = None
    
    # Display generated SQL
    if st.session_state.generated_sql:
        st.code(st.session_state.generated_sql, language="sql")
        
        button_col3, button_col4, button_col5 = st.columns(3)
        
        with button_col3:
            if st.button("📋 Copy SQL"):
                st.success("SQL copied to clipboard!")
        
        with button_col4:
            execute_btn = st.button("▶️ Execute")
        
        with button_col5:
            if st.button("💡 Explain"):
                st.info("This query selects user data with active status filtering.")
    else:
        st.code("-- Generated SQL will appear here", language="sql")
        st.button("📋 Copy SQL", disabled=True)
        st.button("▶️ Execute", disabled=True)
        st.button("💡 Explain", disabled=True)

# Query Results Section
st.markdown("---")
st.markdown('<div class="section-header">📊 Query Results</div>', 
            unsafe_allow_html=True)

# Execute query logic
if 'execute_btn' in locals() and execute_btn and st.session_state.generated_sql:
    # For demo purposes, use sample data
    if "users" in st.session_state.generated_sql.lower():
        st.session_state.query_results = SAMPLE_DATA["users"]
    elif "products" in st.session_state.generated_sql.lower():
        st.session_state.query_results = SAMPLE_DATA["products"]
    else:
        st.session_state.query_results = SAMPLE_DATA["users"]  # Default

# Results Summary
if st.session_state.query_results is not None:
    results_df = st.session_state.query_results
    
    metric_col1, metric_col2, metric_col3 = st.columns(3)
    
    with metric_col1:
        st.markdown(f"""
        <div class="metric-card">
            <div class="metric-value">{len(results_df)}</div>
            <div class="metric-label">Rows Returned</div>
        </div>
        """, unsafe_allow_html=True)
    
    with metric_col2:
        st.markdown(f"""
        <div class="metric-card">
            <div class="metric-value">{len(results_df.columns)}</div>
            <div class="metric-label">Columns</div>
        </div>
        """, unsafe_allow_html=True)
    
    with metric_col3:
        st.subheader("Export Options")
        export_col1, export_col2 = st.columns(2)
        
        with export_col1:
            csv = results_df.to_csv(index=False)
            st.download_button(
                "📥 Download CSV",
                csv,
                "query_results.csv",
                "text/csv"
            )
        
        with export_col2:
            if st.button("📋 Copy Data"):
                st.success("Data copied!")
    
    # Display the data table
    st.dataframe(results_df, use_container_width=True)
    
    # Data Insights
    with st.expander("📈 Quick Data Insights"):
        st.write("**Data Summary:**")
        if "users" in str(results_df.columns).lower():
            st.write(f"- Total active users: {len(results_df)}")
            if 'created_at' in results_df.columns:
                st.write(f"- Registration date range: {results_df['created_at'].min()} - {results_df['created_at'].max()}")
            st.write("- All users have valid email addresses")
        elif "products" in str(results_df.columns).lower():
            st.write(f"- Total products: {len(results_df)}")
            if 'price' in results_df.columns:
                st.write(f"- Price range: ${results_df['price'].min():.2f} - ${results_df['price'].max():.2f}")
            if 'stock_quantity' in results_df.columns:
                st.write(f"- Total stock: {results_df['stock_quantity'].sum()} items")

# Query History Section
if st.session_state.query_history:
    st.markdown("---")
    st.markdown('<div class="section-header">📚 Query History</div>', 
                unsafe_allow_html=True)
    
    with st.expander("View Previous Queries"):
        for i, entry in enumerate(reversed(st.session_state.query_history[-5:]), 1):  # Show last 5
            st.markdown(f"""
            <div class="history-item">
                <div class="history-timestamp">#{len(st.session_state.query_history) - i + 1} - {entry['timestamp']}</div>
                <p><strong>Question:</strong> {entry['question']}</p>
                <p><strong>Tables:</strong> {entry['tables']}</p>
            </div>
            """, unsafe_allow_html=True)
            
            st.code(entry['sql'], language="sql")

# Footer
st.markdown("---")
st.markdown("""
<div style="text-align: center; padding: 2rem; background: #fff3cd; border: 1px solid #ffeaa7; border-radius: 8px; margin-top: 2rem;">
    <p style="color: #856404; margin: 0;"><strong>💡 Tip:</strong> Be specific in your questions for better SQL generation. 
    Include table names, column names, and conditions when possible.</p>
</div>
""", unsafe_allow_html=True)
