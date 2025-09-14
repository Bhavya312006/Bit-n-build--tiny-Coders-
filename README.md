import streamlit as st
import pandas as pd
import plotly.express as px
import os
from io import BytesIO

# --------------------------
# Load Your Dataset
# --------------------------
df = pd.read_csv("synthetic_budget_flow_dataset.csv")

# --------------------------
# Multi-Currency Support
# --------------------------
st.sidebar.header("💱 Currency Options")
currency = st.sidebar.radio("Select Currency", ["INR", "USD"])
conversion_rate = 0.012  # Example: 1 INR = 0.012 USD

def convert_currency(amount):
    return amount * conversion_rate if currency == "USD" else amount

currency_symbol = "$" if currency == "USD" else "₹"

# Convert budget columns for display
df_display = df.copy()
df_display["Budget_Allocated"] = df_display["Budget_Allocated"].apply(convert_currency)
df_display["Budget_Spent"] = df_display["Budget_Spent"].apply(convert_currency)

# --------------------------
# Streamlit Page Setup
# --------------------------
st.set_page_config(page_title="Budget & Expense Tracker", page_icon="💰", layout="wide")
st.title("💰 Budget & Expense Dashboard")
st.markdown("### Track departmental spending, detect anomalies & gather feedback")

# --------------------------
# Sidebar - Filters
# --------------------------
st.sidebar.header("🔎 Filters")
department_filter = st.sidebar.multiselect("Select Department", options=df_display["Department"].unique(), default=df_display["Department"].unique())
vendor_filter = st.sidebar.multiselect("Select Vendor", options=df_display["Vendor"].unique(), default=df_display["Vendor"].unique())
search_text = st.sidebar.text_input("Search Vendor/Department")

filtered_df = df_display[df_display["Department"].isin(department_filter) & df_display["Vendor"].isin(vendor_filter)]
if search_text:
    filtered_df = filtered_df[filtered_df.apply(lambda row: search_text.lower() in row.to_string().lower(), axis=1)]

# --------------------------
# Anomaly Detection
# --------------------------
filtered_df["OverBudget"] = filtered_df["Budget_Spent"] > filtered_df["Budget_Allocated"]

# --------------------------
# Layout: Charts & Table
# --------------------------
col1, col2 = st.columns(2)

with col1:
    st.subheader("📊 Department-wise Spending")
    dept_summary = filtered_df.groupby("Department", as_index=False)["Budget_Spent"].sum()
    dept_chart = px.bar(
        dept_summary,
        x="Department", y="Budget_Spent", color="Department",
        text="Budget_Spent", template="plotly_white",
        color_discrete_sequence=px.colors.qualitative.Vivid
    )
    dept_chart.update_traces(texttemplate=currency_symbol + "%{text:,.0f}", textposition="outside")
    dept_chart.update_layout(xaxis_title="Department", yaxis_title=f"Budget Spent ({currency})", showlegend=False)
    st.plotly_chart(dept_chart, use_container_width=True)

with col2:
    st.subheader("🍩 Vendor Spend Distribution")
    vendor_chart = px.pie(
        filtered_df, values="Budget_Spent", names="Vendor",
        title="Spending by Vendor", hole=0.4,
        color_discrete_sequence=px.colors.qualitative.Set2
    )
    vendor_chart.update_layout(legend_title="Vendor", legend=dict(orientation="v", y=1, x=1))
    st.plotly_chart(vendor_chart, use_container_width=True)

# --------------------------
# Budget vs Actual - Clean Horizontal Grouped Bar Chart
# --------------------------
st.subheader(f"📊 Budget vs Actual Spend (Per Vendor) [{currency}]")
comparison_df = filtered_df.melt(
    id_vars=["Vendor"], 
    value_vars=["Budget_Spent", "Budget_Allocated"], 
    var_name="Type", value_name="Value"
)

budget_chart = px.bar(
    comparison_df, x="Value", y="Vendor", color="Type", barmode="group",
    orientation="h", text="Value", template="plotly_white",
    color_discrete_map={"Budget_Spent": "#EF553B", "Budget_Allocated": "#00CC96"}
)

budget_chart.update_traces(
    texttemplate=currency_symbol + "%{text:,.0f}", 
    textposition="inside", 
    insidetextanchor="start",
    textfont=dict(size=12)
)

budget_chart.update_layout(
    xaxis_title=f"Amount ({currency})", 
    yaxis_title="Vendor", 
    legend_title="",
    bargap=0.4,
    barmode="group",
    height=600
)
st.plotly_chart(budget_chart, use_container_width=True)

# --------------------------
# Highlight anomalies
# --------------------------
st.subheader("⚠ Anomaly Detection - Budget Overruns")
anomalies = filtered_df[filtered_df["OverBudget"]]
if not anomalies.empty:
    st.error("🚨 Budget Overruns Detected!")
    st.dataframe(anomalies, use_container_width=True)
else:
    st.success("✅ No budget overruns detected.")

# --------------------------
# Feedback System with Persistent Storage
# --------------------------
st.subheader("💭 Community Feedback & Ratings")
feedback = st.text_area("Leave your feedback or suggestions here:")

st.markdown("### ⭐ Rate the Dashboard")
rating_options = ["⭐", "⭐⭐", "⭐⭐⭐", "⭐⭐⭐⭐", "⭐⭐⭐⭐⭐"]
rating = st.radio("Select your rating:", rating_options, horizontal=True)
rating_value = rating_options.index(rating) + 1

# Load feedback from file OR session
if "feedback_data" not in st.session_state:
    if os.path.exists("feedback.csv"):
        st.session_state.feedback_data = pd.read_csv("feedback.csv")
    else:
        st.session_state.feedback_data = pd.DataFrame(columns=["Feedback", "Rating"])

# Add new feedback
if st.button("Submit Feedback"):
    if feedback.strip():
        new_feedback = pd.DataFrame([[feedback, rating_value]], columns=["Feedback", "Rating"])
        st.session_state.feedback_data = pd.concat([st.session_state.feedback_data, new_feedback], ignore_index=True)
        st.session_state.feedback_data.to_csv("feedback.csv", index=False)
        st.success(f"✅ Thank you! Your feedback was saved with {rating_value}⭐")
    else:
        st.warning("⚠ Feedback cannot be empty.")

# Display feedback immediately
st.subheader("📋 Community Feedback Received")
st.dataframe(st.session_state.feedback_data, use_container_width=True)

# Option to download CSV
if not st.session_state.feedback_data.empty:
    csv_buffer = BytesIO()
    st.session_state.feedback_data.to_csv(csv_buffer, index=False)
    csv_buffer.seek(0)
    st.download_button("📥 Download Feedback CSV", data=csv_buffer, file_name="feedback.csv", mime="text/csv")

# --------------------------
# Chatbot Helper
# --------------------------
st.subheader("🤖 Budget Chatbot Helper")
user_query = st.text_input("Ask a question about the budget or vendors:")
if st.button("Ask"):
    response = "Sorry, I could not understand your query."
    query_lower = user_query.lower()
    
    if "department" in query_lower:
        dept_summary = filtered_df.groupby("Department")["Budget_Spent"].sum()
        response = "Department-wise Spending:\n" + "\n".join([f"{d}: {currency_symbol}{v:,.0f}" for d,v in dept_summary.items()])
    elif "vendor" in query_lower:
        vendor_summary = filtered_df.groupby("Vendor")["Budget_Spent"].sum()
        response = "Vendor-wise Spending:\n" + "\n".join([f"{v}: {currency_symbol}{val:,.0f}" for v,val in vendor_summary.items()])
    elif "overbudget" in query_lower or "anomaly" in query_lower:
        overbudget_count = filtered_df["OverBudget"].sum()
        response = f"There are {overbudget_count} budget overruns."
    
    st.info(response)

# --------------------------
# Final Transactions Table
# --------------------------
st.subheader("📂 Filtered Transactions")
st.dataframe(filtered_df, use_container_width=True)
