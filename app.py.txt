
import streamlit as st
import pandas as pd
import os, re
from datetime import datetime, date, timedelta
import plotly.express as px
from database import init_db, get_conn, q, fetch_one, fetch_all, log_action, DB_FILE
from finance import project_stats, company_stats
from advisor import advisor_text, scenario_calc
from whatsapp import wa_link, quote_message, variation_message, payment_reminder, invoice_message, clean_sa_number
from pdfs import quote_pdf, invoice_pdf, variation_pdf, project_report_pdf
from utils import money, parse_whatsapp_expenses

st.set_page_config(page_title="Nova Vital Advisor OS", page_icon="💎", layout="wide", initial_sidebar_state="expanded")
init_db()

# --- SECURITY ---
APP_PIN = os.getenv("NOVA_PIN", "1234")
if "auth" not in st.session_state:
    st.session_state.auth=False

def check_pin():
    st.title("💎 Nova Vital Advisor OS")
    st.subheader("Contractor ERP | Secure Login")
    pin = st.text_input("Enter PIN", type="password")
    if st.button("Unlock"):
        if pin==APP_PIN:
            st.session_state.auth=True
            log_action("LOGIN","auth","owner","successful login")
            st.rerun()
        else:
            st.error("Invalid PIN")
    st.stop()

if not st.session_state.auth:
    check_pin()

# --- SIDEBAR NAV ---
st.sidebar.title("💎 Nova Vital OS")
pages = ["🏠 Executive Dashboard","📁 Projects","👥 Clients","💰 Cash Flow","💳 Payments","📑 Quotes","🧾 Invoices","🔄 Variations","🛒 Procurement","📦 Inventory","👷 Workers","✅ Tasks & Snags","📓 Site Diary","📈 Leads","🤖 AI Advisor","📊 Reports","⚙️ Settings"]
page = st.sidebar.radio("Navigation", pages, label_visibility="collapsed")

# Helper get projects
def get_projects_df():
    conn=get_conn()
    df=pd.read_sql_query("SELECT * FROM projects ORDER BY created_at DESC", conn)
    conn.close()
    return df

projects_df = get_projects_df()
project_options = {f"{r['name']} ({r['status']})": int(r['id']) for _, r in projects_df.iterrows()} if not projects_df.empty else {}

def delete_project_cascade(pid):
    conn=get_conn()
    cur=conn.cursor()
    cur.execute("DELETE FROM projects WHERE id=?", (pid,))
    conn.commit()
    conn.close()
    log_action("DELETE","projects",pid,"hard deleted")


# --- DASHBOARD ---
if page=="🏠 Executive Dashboard":
    st.title("🏠 Executive Dashboard")
    stats = company_stats()
    df = stats["df"]
    if df.empty:
        st.info("No projects yet. Create your first project in Projects page. Example: Bluevalley (Kitchen) Material R50 000 Labor 40%")
        st.stop()
    c1,c2,c3,c4,c5,c6 = st.columns(6)
    c1.metric("Active Projects", len(df[df["status"]=="Active"]))
    c2.metric("Total Contract", money(stats["total_contract"]))
    c3.metric("Approved Variations", money(df["approved_variations"].sum()))
    c4.metric("Received", money(stats["total_received"]))
    c5.metric("Outstanding", money(stats["total_outstanding"]))
    c6.metric("Projected Profit", money(stats["total_profit"]), f"{stats['avg_margin']:.1f}% avg")

    # Risk warnings
    risks=[]
    for _, r in df.iterrows():
        if r["margin"]<20 and r["revenue"]>0: risks.append(f"🔴 {r['name']}: Low margin {r['margin']:.1f}%")
        if r["outstanding"]> r["revenue"]*0.3 and r["outstanding"]>5000: risks.append(f"🟡 {r['name']}: High outstanding R{r['outstanding']:,.0f}")
        if r["spent"]> r["total_budget"]*1.1 and r["total_budget"]>0: risks.append(f"🔴 {r['name']}: Over budget")
    if risks:
        st.warning("\n".join(risks[:10]))

    col1,col2 = st.columns(2)
    with col1:
        fig=px.bar(df, x="name", y="profit", color="margin", title="Profit by Project", color_continuous_scale="Viridis")
        st.plotly_chart(fig, use_container_width=True)
    with col2:
        fig=px.bar(df, x="name", y="outstanding", title="Outstanding Payments")
        st.plotly_chart(fig, use_container_width=True)

    col3,col4 = st.columns(2)
    with col3:
        # expenses by category
        conn=get_conn()
        exp=pd.read_sql_query("SELECT category, SUM(amount) as total FROM expenses GROUP BY category", conn)
        conn.close()
        if not exp.empty:
            fig=px.pie(exp, values="total", names="category", title="Expenses by Category")
            st.plotly_chart(fig, use_container_width=True)
    with col4:
        # monthly revenue vs expenses
        conn=get_conn()
        monthly_exp=pd.read_sql_query("SELECT substr(expense_date,1,7) as month, SUM(amount) as total FROM expenses GROUP BY month", conn)
        monthly_pay=pd.read_sql_query("SELECT substr(payment_date,1,7) as month, SUM(amount) as total FROM payments GROUP BY month", conn)
        conn.close()
        if not monthly_exp.empty or not monthly_pay.empty:
            merged=pd.merge(monthly_exp, monthly_pay, on="month", how="outer", suffixes=("_exp","_pay")).fillna(0)
            fig=px.line(merged, x="month", y=["total_exp","total_pay"], title="Monthly Cash In vs Out")
            st.plotly_chart(fig, use_container_width=True)

    st.subheader("📊 Company P&L Snapshot")
    st.dataframe(df[["name","status","revenue","received","outstanding","spent","profit","margin"]], use_container_width=True)

# --- PROJECTS ---
elif page=="📁 Projects":
    st.title("📁 Project Management")
    tab_list, tab_create = st.tabs(["📋 Active Journals", "➕ Create New"])
    with tab_create:
        # --- CLIENT SELECTOR FROM CLIENT BOOK ---
        conn=get_conn()
        clients_df=pd.read_sql_query("SELECT * FROM clients ORDER BY name", conn)
        conn.close()
        client_names = ["➕ New Client"] + clients_df["name"].tolist() if not clients_df.empty else ["➕ New Client"]
        sel_client = st.selectbox("📇 Select from Client Book (Phonebook)", client_names, help="Pick existing client to auto-fill, or add new")
        pre_name, pre_phone, pre_email, pre_addr = "", "", "", ""
        pre_client_id = None
        if sel_client != "➕ New Client" and not clients_df.empty:
            crow = clients_df[clients_df["name"]==sel_client].iloc[0]
            pre_name = crow["name"]
            pre_phone = crow["phone"] or ""
            pre_email = crow["email"] or ""
            pre_addr = crow["address"] or ""
            pre_client_id = int(crow["id"])
            st.info(f"Using Client Book: {pre_name} | {pre_phone}")

        with st.form("create_proj"):
            a,b,c = st.columns(3)
            name=a.text_input("Project Name*", placeholder="Bluevalley (Kitchen)")
            client_name=b.text_input("Client Name", value=pre_name)
            phone=c.text_input("Client WhatsApp", value=pre_phone, placeholder="0821234567")
            d,e,f = st.columns(3)
            email=d.text_input("Client Email", value=pre_email)
            address=e.text_input("Site Address", value=pre_addr)
            status=f.selectbox("Status", ["Lead","Quoted","Approved","Active","On Hold","Completed","Cancelled"], index=3)
            st.markdown("### 💰 Automated Pricing Model (Labor 40% + Hardware Rebates)")
            g,h,i,j,k = st.columns(5)
            material=g.number_input("Material Budget R", value=50000.0, step=1000.0, help="Total hardware/boards/tops")
            labor_pct=h.number_input("Labor % (auto 40%)", value=40.0, min_value=0.0, max_value=200.0, help="Labor = Material x %")
            rebate_pct=i.number_input("Hardware Rebate %", value=5.0, min_value=0.0, max_value=50.0, help="Rebate from supplier, e.g. 5% back")
            transport=j.number_input("Transport Budget R", value=2000.0, step=100.0)
            markup=k.number_input("Markup %", value=25.0, help="Markup on (Material + Labor + Transport)")

            # Automated calcs
            labor_auto = material * labor_pct / 100
            rebate_auto = material * rebate_pct / 100
            effective_material = material - rebate_auto
            cost_before_markup = effective_material + labor_auto + transport
            cost_without_rebate = material + labor_auto + transport
            contract_auto = cost_without_rebate * (1+markup/100)
            profit_auto = contract_auto - cost_before_markup + rebate_auto  # rebate adds to profit
            # Actually profit = contract - effective cost, but rebate is profit boost
            profit_auto_simple = contract_auto - cost_before_markup

            c1,c2,c3,c4 = st.columns(4)
            c1.metric("Labor (40%)", f"R{labor_auto:,.0f}")
            c2.metric("Rebate Income", f"R{rebate_auto:,.0f}", f"{rebate_pct}% of material")
            c3.metric("Effective Material", f"R{effective_material:,.0f}")
            c4.metric("Projected Profit", f"R{profit_auto_simple:,.0f}", f"{(profit_auto_simple/contract_auto*100 if contract_auto else 0):.1f}%")

            with st.expander("See full breakdown"):
                st.write(f"Material: R{material:,.2f}")
                st.write(f"Labor ({labor_pct}% of material): R{labor_auto:,.2f}")
                st.write(f"Rebate ({rebate_pct}%): -R{rebate_auto:,.2f} (goes to profit)")
                st.write(f"Effective Material Cost: R{effective_material:,.2f}")
                st.write(f"Transport: R{transport:,.2f}")
                st.write(f"Total Cost (with rebate benefit): R{cost_before_markup:,.2f}")
                st.write(f"Contract (Material+Labor+Transport + {markup}% markup): R{contract_auto:,.2f}")
                st.write(f"Profit = Contract - Effective Cost = R{profit_auto_simple:,.2f}")

            contract=g.number_input("Contract Value R (auto-calc, you can override)", value=contract_auto, step=500.0, key="contract_final")
            notes=st.text_area("Notes")
            start_date=st.date_input("Start Date", date.today())
            due_date=st.date_input("Due Date", date.today()+timedelta(days=30))
            if st.form_submit_button("Create Project"):
                if not name:
                    st.error("Name required")
                else:
                    try:
                        q("""INSERT INTO projects
                            (name, client_id, client_name, client_phone, client_email, address, material_budget, labor_pct, transport_budget, markup_pct, contract_value, status, notes, start_date, due_date, created_at, updated_at, rebate_pct, rebate_value, labor_auto_calc)
                            VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)""",
                          (name, pre_client_id, client_name, phone, email, address, material, labor_pct, transport, markup, contract, status, notes, str(start_date), str(due_date), datetime.now().isoformat(), datetime.now().isoformat(), rebate_pct, rebate_auto, 1))
                        log_action("CREATE","project",name,f"Created {name}")
                        st.success(f"Created {name} | Contract R{contract:,.0f} Labour R{material*labor_pct/100:,.0f}")
                        st.rerun()
                    except Exception as ex:
                        st.error(f"Error: {ex} - Maybe project name exists?")

    with tab_list:
        if projects_df.empty:
            st.info("No projects")
        else:
            for _, proj_row in projects_df.iterrows():
                pid=int(proj_row["id"])
                s=project_stats(pid)
                with st.expander(f"{proj_row['name']} | {proj_row['status']} | Contract {money(proj_row['contract_value'])} | Profit {money(s['profit'])} ({s['margin']:.1f}%)", expanded=False):
                    tabs = st.tabs(["Overview","Financials","Expenses","Payments","Variations","Procurement","Tasks","Snags","Site Diary","Reports"])
                    with tabs[0]:
                        c1,c2 = st.columns(2)
                        c1.write(f"**Client:** {proj_row['client_name']} | {proj_row['client_phone']} | {proj_row['client_email']}")
                        c1.write(f"**Site:** {proj_row['address']}")
                        c1.write(f"**Start:** {proj_row['start_date']} Due: {proj_row['due_date']}")
                        c2.write(f"Material Budget: {money(s['material_budget'])} | Labour Budget: {money(s['labor_budget'])} ({s['labor_pct']}%) | Transport: {money(s['transport_budget'])}")
                        c2.write(f"Revenue: {money(s['revenue'])} | Received: {money(s['received'])} | Outstanding: {money(s['outstanding'])}")
                        st.markdown(advisor_text(pid))

                    with tabs[1]:
                        st.metric("Projected Profit", money(s["profit"]), f"{s['margin']:.1f}%")
                        df_fin = pd.DataFrame([{"Item":k,"Amount":v} for k,v in {
                            "Contract": s["contract_value"], "Approved Variations": s["approved_variations"],
                            "Revenue": s["revenue"], "Received": s["received"], "Outstanding": s["outstanding"],
                            "Material Actual": s["material"], "Labor Actual": s["labor"], "Transport Actual": s["transport"],
                            "Total Spent": s["spent"], "Profit": s["profit"]
                        }.items()])
                        st.dataframe(df_fin, use_container_width=True)

                    with tabs[2]:
                        # WhatsApp paste + expense form
                        t1,t2 = st.tabs(["Add Expense","Paste WhatsApp"])
                        with t1:
                            with st.form(f"exp_{pid}", clear_on_submit=True):
                                ca,cb = st.columns(2)
                                ed=ca.date_input("Date", date.today())
                                cat=cb.selectbox("Category", ["Board/Melamine","Hardware & Runners","Quartz/Tops","Labor Pay","Transport/Fuel","Sundries","Rebate","Hardware Rebate","Other"])
                                desc=st.text_input("Description", placeholder="e.g. Thabo 2 days + extra handles")
                                cc,cd,ce = st.columns(3)
                                amt=cc.number_input("Amount R", min_value=0.0, step=50.0)
                                sup=cd.text_input("Supplier")
                                un=ce.checkbox("Unplanned?")
                                bill_status=st.selectbox("Billable Status", ["Non-Billable","Potentially Billable","Billable"], index=2 if un else 0)
                                if st.form_submit_button("Add"):
                                    if amt>0:
                                        q("INSERT INTO expenses (project_id, expense_date, category, description, amount, supplier, is_unplanned, billable_status, created_at) VALUES (?,?,?,?,?,?,?,?,?)",
                                          (pid, str(ed), cat, desc, amt, sup, 1 if un else 0, bill_status, datetime.now().isoformat()))
                                        log_action("CREATE","expense",pid,f"{cat} R{amt}")
                                        st.rerun()
                        with t2:
                            wa=st.text_area("Paste WhatsApp", placeholder="[10:23] Thabo: Bought boards R5500\nDiesel R650", height=120, key=f"wa_{pid}")
                            if st.button("AI Extract", key=f"ai_{pid}"):
                                parsed=parse_whatsapp_expenses(wa)
                                for p in parsed:
                                    q("INSERT INTO expenses (project_id, expense_date, category, description, amount, is_unplanned, billable_status, created_at) VALUES (?,?,?,?,?,?,?,?)",
                                      (pid, str(date.today()), p["category"], p["description"], p["amount"], p["is_unplanned"], p["billable_status"], datetime.now().isoformat()))
                                st.success(f"Extracted {len(parsed)} expenses")
                                st.rerun()
                        conn=get_conn()
                        df_exp=pd.read_sql_query("SELECT * FROM expenses WHERE project_id=? ORDER BY expense_date DESC", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_exp, use_container_width=True)

                    with tabs[3]:
                        with st.form(f"pay_{pid}", clear_on_submit=True):
                            pa,pb,pc = st.columns(3)
                            pd_date=pa.date_input("Payment Date", date.today())
                            amt=pb.number_input("Amount", min_value=0.0, step=100.0)
                            ptype=pc.selectbox("Type", ["Deposit","Progress Payment","Stage Payment","Final Payment","Other"])
                            meth=st.selectbox("Method", ["EFT","Cash","Card"])
                            ref=st.text_input("Reference")
                            desc=st.text_input("Description")
                            if st.form_submit_button("Add Payment"):
                                if amt>0:
                                    q("INSERT INTO payments (project_id, payment_date, amount, payment_type, method, reference, description, created_at) VALUES (?,?,?,?,?,?,?,?)",
                                      (pid, str(pd_date), amt, ptype, meth, ref, desc, datetime.now().isoformat()))
                                    st.rerun()
                        conn=get_conn()
                        df_pay=pd.read_sql_query("SELECT * FROM payments WHERE project_id=?", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_pay, use_container_width=True)
                        if not df_pay.empty:
                            st.info(f"Received {money(s['received'])} Outstanding {money(s['outstanding'])}")

                    with tabs[4]:
                        with st.form(f"var_{pid}", clear_on_submit=True):
                            va,vb = st.columns(2)
                            vdesc=va.text_input("Variation Description*")
                            vdate=vb.date_input("Date", date.today())
                            vc,vd,ve,vf = st.columns(4)
                            mat_c=vc.number_input("Material Cost", min_value=0.0, step=100.0)
                            lab_c=vd.number_input("Labour Cost", min_value=0.0, step=100.0)
                            tr_c=ve.number_input("Transport Cost", min_value=0.0, step=100.0)
                            mk=vf.number_input("Markup %", value=25.0)
                            cost_p = mat_c+lab_c+tr_c
                            client_p = cost_p * (1+mk/100)
                            st.write(f"Cost R{cost_p:,.0f} -> Client Price R{client_p:,.0f}")
                            if st.form_submit_button("Create Variation"):
                                var_no=f"VAR-{datetime.now().strftime('%Y%m%d')}-{pid}-{int(datetime.now().timestamp())%10000}"
                                q("""INSERT INTO variations
                                    (project_id, variation_no, variation_date, description, material_cost, labor_cost, transport_cost, markup_pct, cost_price, client_price, status, created_at)
                                    VALUES (?,?,?,?,?,?,?,?,?,?,?,?)""",
                                  (pid, var_no, str(vdate), vdesc, mat_c, lab_c, tr_c, mk, cost_p, client_p, "Pending", datetime.now().isoformat()))
                                st.success(f"Created {var_no}")
                                st.rerun()
                        conn=get_conn()
                        df_var=pd.read_sql_query("SELECT * FROM variations WHERE project_id=? ORDER BY id DESC", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_var, use_container_width=True)
                        if not df_var.empty:
                            for _, vr in df_var.iterrows():
                                if vr["status"]!="Approved":
                                    if st.button(f"Approve {vr['variation_no']}", key=f"appr_{vr['id']}"):
                                        q("UPDATE variations SET status='Approved' WHERE id=?", (int(vr["id"]),))
                                        log_action("APPROVE","variation",vr["variation_no"],f"Approved R{vr['client_price']}")
                                        st.rerun()
                                wa_msg = variation_message(proj_row["name"], proj_row["client_name"] or "Client", vr["variation_no"], vr["client_price"], vr["description"])
                                st.markdown(f"[💬 WhatsApp Approval for {vr['variation_no']}]({wa_link(proj_row['client_phone'], wa_msg)})")

                    with tabs[5]:
                        with st.form(f"proc_{pid}", clear_on_submit=True):
                            pa,pb,pc = st.columns(3)
                            item=pa.text_input("Item")
                            qty=pb.number_input("Qty", value=1.0)
                            unit=pc.text_input("Unit", value="pcs")
                            pd_,pe,pf = st.columns(3)
                            sup=pd_.text_input("Supplier")
                            quoted=pe.number_input("Quoted Price", min_value=0.0)
                            actual=pf.number_input("Actual Price", min_value=0.0)
                            if st.form_submit_button("Add Procurement"):
                                q("INSERT INTO procurement (project_id, item, quantity, unit, supplier, quoted_price, actual_price, status, created_at) VALUES (?,?,?,?,?,?,?, ?,?)",
                                  (pid, item, qty, unit, sup, quoted, actual, "Required", datetime.now().isoformat()))
                                st.rerun()
                        conn=get_conn()
                        df_proc=pd.read_sql_query("SELECT * FROM procurement WHERE project_id=?", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_proc, use_container_width=True)

                    with tabs[6]:
                        with st.form(f"task_{pid}", clear_on_submit=True):
                            ta,tb = st.columns(2)
                            title=ta.text_input("Task Title")
                            due=tb.date_input("Due", date.today())
                            tc,td = st.columns(2)
                            assign=tc.text_input("Assigned To")
                            prio=td.selectbox("Priority", ["Low","Medium","High"])
                            if st.form_submit_button("Add Task"):
                                q("INSERT INTO tasks (project_id, title, due_date, assigned_to, priority, status, created_at) VALUES (?,?,?,?,?,?,?)",
                                  (pid, title, str(due), assign, prio, "Pending", datetime.now().isoformat()))
                                st.rerun()
                        conn=get_conn()
                        df_task=pd.read_sql_query("SELECT * FROM tasks WHERE project_id=?", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_task, use_container_width=True)

                    with tabs[7]:
                        with st.form(f"snag_{pid}", clear_on_submit=True):
                            sa,sb = st.columns(2)
                            desc=sa.text_input("Snag Description")
                            prio=sb.selectbox("Priority", ["Low","Medium","High","Critical"])
                            sc,sd = st.columns(2)
                            assign=sc.text_input("Assigned")
                            due=sd.date_input("Due", date.today())
                            if st.form_submit_button("Add Snag"):
                                q("INSERT INTO snags (project_id, description, priority, assigned_to, due_date, status, created_at) VALUES (?,?,?,?,?,?,?)",
                                  (pid, desc, prio, assign, str(due), "Open", datetime.now().isoformat()))
                                st.rerun()
                        conn=get_conn()
                        df_snag=pd.read_sql_query("SELECT * FROM snags WHERE project_id=?", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_snag, use_container_width=True)

                    with tabs[8]:
                        with st.form(f"diary_{pid}", clear_on_submit=True):
                            da,db = st.columns(2)
                            ld=da.date_input("Log Date", date.today())
                            workers=db.text_input("Workers Present", placeholder="Thabo, Peter")
                            work=st.text_area("Work Done")
                            mat_used=st.text_area("Materials Used")
                            issues=st.text_area("Issues / Delays")
                            notes=st.text_area("Notes")
                            if st.form_submit_button("Save Diary"):
                                q("INSERT INTO site_logs (project_id, log_date, workers, work_done, materials_used, issues, notes, created_at) VALUES (?,?,?,?,?,?,?,?)",
                                  (pid, str(ld), workers, work, mat_used, issues, notes, datetime.now().isoformat()))
                                st.rerun()
                        conn=get_conn()
                        df_logs=pd.read_sql_query("SELECT * FROM site_logs WHERE project_id=? ORDER BY log_date DESC", conn, params=(pid,))
                        conn.close()
                        st.dataframe(df_logs, use_container_width=True)

                    with tabs[9]:
                        if st.button(f"Generate Full PDF - {proj_row['name']}", key=f"pdf_{pid}"):
                            conn=get_conn()
                            df_exp=pd.read_sql_query("SELECT * FROM expenses WHERE project_id=?", conn, params=(pid,))
                            conn.close()
                            path=project_report_pdf(proj_row, s, df_exp)
                            with open(path,"rb") as f:
                                st.download_button("Download Report", f, file_name=f"{proj_row['name']}_Report.pdf")

# --- CASH FLOW ---

    # --- GLOBAL DELETE - ALWAYS VISIBLE IN PROJECTS PAGE ---
    st.divider()
    with st.expander("⚠️ Danger Zone - Delete Project (Permanent)", expanded=False):
        if not project_options:
            st.info("No projects to delete")
        else:
            del_sel = st.selectbox("Select project to delete permanently", list(project_options.keys()), key="del_sel_global")
            pid_tmp = project_options[del_sel]
            conn=get_conn()
            exp_c = conn.execute("SELECT COUNT(*) FROM expenses WHERE project_id=?", (pid_tmp,)).fetchone()[0]
            pay_c = conn.execute("SELECT COUNT(*) FROM payments WHERE project_id=?", (pid_tmp,)).fetchone()[0]
            var_c = conn.execute("SELECT COUNT(*) FROM variations WHERE project_id=?", (pid_tmp,)).fetchone()[0]
            conn.close()
            st.warning(f"This will PERMANENTLY delete **{del_sel}** + {exp_c} expenses + {pay_c} payments + {var_c} variations + tasks/logs. CANNOT be undone.")
            confirm = st.text_input(f'Type DELETE to confirm deletion of {del_sel}', key="confirm_del_global")
            if st.button("🔴 Permanently Delete Project", type="primary", key="btn_del_global"):
                if confirm == "DELETE":
                    delete_project_cascade(pid_tmp)
                    st.success(f"Deleted {del_sel}")
                    st.rerun()
                else:
                    st.error("Type DELETE exactly to confirm")


elif page=="👥 Clients":
    st.title("👥 Client Book - Phonebook Integration")
    st.caption("Import from your phone's contacts in 1 tap (Android Chrome), or upload CSV/Google Contacts export")

    # --- CONTACT PICKER (Phone Book) ---
    st.subheader("📱 Import from Phone Book (Android)")
    st.markdown("**On your phone:** Tap the button below → Select contacts → They will appear as a table to import")
    
    import streamlit.components.v1 as components
    contact_picker_html = """
    <div style="font-family:sans-serif">
      <button id="pickBtn" style="background:#0f6fff;color:white;padding:12px 18px;border:none;border-radius:10px;font-weight:600;cursor:pointer;width:100%">📇 Open Phone Book</button>
      <div id="status" style="margin-top:10px;color:#666;font-size:13px"></div>
      <div id="out" style="margin-top:12px;max-height:300px;overflow:auto;border:1px solid #ddd;border-radius:8px;padding:8px;display:none"></div>
      <textarea id="jsonOut" style="width:100%;height:120px;margin-top:10px;display:none" placeholder="Contacts JSON will appear here - copy and paste into the box below in Streamlit"></textarea>
    </div>
    <script>
    const btn = document.getElementById('pickBtn');
    const status = document.getElementById('status');
    const out = document.getElementById('out');
    const jsonOut = document.getElementById('jsonOut');
    btn.addEventListener('click', async () => {
      if (!('contacts' in navigator && 'ContactsManager' in window)) {
        status.innerHTML = '❌ Your browser does not support Contact Picker. On iPhone use CSV export, on Android use Chrome. You can still upload CSV below.';
        return;
      }
      try {
        const props = ['name','tel','email','address'];
        const opts = {multiple: true};
        status.innerHTML = 'Opening phone book...';
        const contacts = await navigator.contacts.select(props, opts);
        if (!contacts.length) { status.innerHTML='No contacts selected'; return; }
        status.innerHTML = `✅ Selected ${contacts.length} contact(s). Copy the JSON below into the Import box in Streamlit.`;
        let html = '<table style="width:100%;font-size:12px"><tr><th>Name</th><th>Phone</th><th>Email</th></tr>';
        let data = [];
        contacts.forEach(c => {
          const name = c.name ? c.name.join(' ') : '';
          const tel = c.tel ? c.tel[0] : '';
          const email = c.email ? c.email[0] : '';
          const addr = c.address ? c.address.map(a=>a.addressLine ? a.addressLine.join(', ') : '').join('; ') : '';
          data.push({name:name, phone:tel, email:email, address:addr});
          html += `<tr><td>${name}</td><td>${tel}</td><td>${email}</td></tr>`;
        });
        html += '</table>';
        out.innerHTML = html;
        out.style.display='block';
        jsonOut.style.display='block';
        jsonOut.value = JSON.stringify(data, null, 2);
      } catch (e) {
        status.innerHTML = '❌ Error: ' + e.message;
      }
    });
    </script>
    """
    components.html(contact_picker_html, height=450)

    st.divider()
    col1, col2 = st.columns(2)
    with col1:
        st.subheader("📥 Paste Phonebook JSON")
        json_input = st.text_area("Paste JSON from Phone Book button above", placeholder='[{"name":"John Dlamini","phone":"+27..."}]', height=150)
        if st.button("Import Pasted Contacts"):
            try:
                import json as js
                data = js.loads(json_input)
                conn=get_conn()
                added=0
                for c in data:
                    name = c.get("name","").strip()
                    if not name: continue
                    phone = c.get("phone","") or c.get("tel","")
                    email = c.get("email","")
                    addr = c.get("address","")
                    # avoid duplicates by phone or name
                    exists = conn.execute("SELECT id FROM clients WHERE phone=? OR name=?", (phone, name)).fetchone()
                    if not exists:
                        conn.execute("INSERT INTO clients (name, phone, email, address, created_at) VALUES (?,?,?,?,?)",
                                     (name, phone, email, addr, datetime.now().isoformat()))
                        added+=1
                conn.commit()
                conn.close()
                st.success(f"Imported {added} new clients from phone book")
                st.rerun()
            except Exception as e:
                st.error(f"Invalid JSON: {e}")

    with col2:
        st.subheader("📤 Upload CSV / Google Contacts")
        st.caption("Export from Google Contacts → Export → Google CSV, or iPhone contacts CSV")
        up = st.file_uploader("Upload CSV", type=["csv"])
        if up:
            try:
                df_up = pd.read_csv(up)
                st.dataframe(df_up.head(), use_container_width=True)
                # try to map columns
                # common Google headers: Name, Given Name, Family Name, Phone 1 - Value, E-mail 1 - Value
                def guess(col_options, df_cols):
                    for o in col_options:
                        for dc in df_cols:
                            if o.lower() in dc.lower():
                                return dc
                    return None
                name_col = guess(["name","display name","given name"], df_up.columns) or df_up.columns[0]
                phone_col = guess(["phone","mobile","tel"], df_up.columns)
                email_col = guess(["email","e-mail"], df_up.columns)
                addr_col = guess(["address","home"], df_up.columns)
                if st.button("Import CSV Contacts"):
                    conn=get_conn()
                    added=0
                    for _, r in df_up.iterrows():
                        name = str(r[name_col]).strip() if name_col and pd.notna(r[name_col]) else ""
                        if not name or name.lower()=="nan": continue
                        phone = str(r[phone_col]) if phone_col and pd.notna(r[phone_col]) else ""
                        email = str(r[email_col]) if email_col and pd.notna(r[email_col]) else ""
                        addr = str(r[addr_col]) if addr_col and pd.notna(r[addr_col]) else ""
                        exists = conn.execute("SELECT id FROM clients WHERE phone=? OR name=?", (phone, name)).fetchone()
                        if not exists:
                            conn.execute("INSERT INTO clients (name, phone, email, address, created_at) VALUES (?,?,?,?,?)",
                                         (name, phone, email, addr, datetime.now().isoformat()))
                            added+=1
                    conn.commit()
                    conn.close()
                    st.success(f"Imported {added} clients from CSV")
                    st.rerun()
            except Exception as e:
                st.error(f"CSV error: {e}")

    st.divider()
    st.subheader("➕ Add Client Manually")
    with st.form("add_client", clear_on_submit=True):
        a,b,c,d = st.columns(4)
        name = a.text_input("Client Name*")
        phone = b.text_input("WhatsApp / Phone*", placeholder="0821234567")
        email = c.text_input("Email")
        company = d.text_input("Company")
        address = st.text_input("Address")
        notes = st.text_area("Notes")
        if st.form_submit_button("Save Client") and name:
            try:
                q("INSERT INTO clients (name, phone, email, address, company, notes, created_at) VALUES (?,?,?,?,?,?,?)",
                  (name, phone, email, address, company, notes, datetime.now().isoformat()))
                st.success(f"Saved {name}")
                st.rerun()
            except Exception as e:
                st.error(f"Error: {e}")

    st.divider()
    conn=get_conn()
    cdf=pd.read_sql_query("SELECT * FROM clients ORDER BY created_at DESC", conn)
    conn.close()
    st.subheader(f"📇 All Clients ({len(cdf)})")
    if cdf.empty:
        st.info("No clients yet. Use phone book button on Android Chrome, or upload CSV.")
    else:
        # search
        search = st.text_input("🔍 Search client", placeholder="Name or phone")
        if search:
            cdf = cdf[cdf["name"].str.contains(search, case=False, na=False) | cdf["phone"].str.contains(search, case=False, na=False)]
        st.dataframe(cdf, use_container_width=True)
        # quick actions
        if not cdf.empty:
            c1,c2 = st.columns(2)
            with c1:
                sel = st.selectbox("Select client for quick project", cdf["name"].tolist())
                if st.button("Create Project for this Client"):
                    st.session_state["prefill_client"] = sel
                    st.info(f"Go to Projects → Create New → {sel} will be pre-selected")
            with c2:
                del_client = st.selectbox("Delete client", [""]+cdf["name"].tolist(), key="del_client")
                if del_client and st.button("Delete Client"):
                    q("DELETE FROM clients WHERE name=?", (del_client,))
                    st.success(f"Deleted {del_client}")
                    st.rerun()

        # Export
        if not cdf.empty:
            csv = cdf.to_csv(index=False).encode()
            st.download_button("Download Client Book CSV", csv, "nova_vital_clients.csv")


elif page=="💰 Cash Flow":
    st.title("💰 Cash Flow")
    stats=company_stats()
    df=stats["df"]
    if df.empty:
        st.info("No data")
        st.stop()
    conn=get_conn()
    pay_df=pd.read_sql_query("SELECT payment_date as date, amount FROM payments", conn)
    exp_df=pd.read_sql_query("SELECT expense_date as date, amount FROM expenses", conn)
    wp_df=pd.read_sql_query("SELECT payment_date as date, amount FROM worker_payments", conn)
    conn.close()
    cash_in = pay_df["amount"].sum() if not pay_df.empty else 0
    cash_out = (exp_df["amount"].sum() if not exp_df.empty else 0) + (wp_df["amount"].sum() if not wp_df.empty else 0)
    closing = cash_in - cash_out
    c1,c2,c3 = st.columns(3)
    c1.metric("Cash In (Received)", money(cash_in))
    c2.metric("Cash Out (Expenses+Wages)", money(cash_out))
    c3.metric("Closing Cash Position", money(closing), delta="Positive" if closing>=0 else "Negative", delta_color="normal" if closing>=0 else "inverse")
    if closing<0:
        st.error("🔴 Cash position negative - prioritize collections")
    if stats["total_outstanding"]>20000:
        st.warning(f"🟡 R{stats['total_outstanding']:,.0f} outstanding from clients - send reminders")
    st.dataframe(df[["name","received","spent","outstanding","profit"]], use_container_width=True)

# --- PAYMENTS, QUOTES, INVOICES etc ---
elif page=="💳 Payments":
    st.title("💳 All Payments")
    conn=get_conn()
    df=pd.read_sql_query("SELECT p.*, pr.name as project_name FROM payments p JOIN projects pr ON p.project_id=pr.id ORDER BY payment_date DESC", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)

elif page=="📑 Quotes":
    st.title("📑 Quotes")
    with st.form("quote_form"):
        a,b = st.columns(2)
        proj_id = a.selectbox("Project", list(project_options.keys()) if project_options else ["No Projects"]) if project_options else None
        valid_until = b.date_input("Valid Until", date.today()+timedelta(days=7))
        desc = st.text_area("Project Description")
        c,d,e = st.columns(3)
        markup = c.number_input("Markup %", value=25.0)
        vat = d.number_input("VAT %", value=0.0)
        discount = e.number_input("Discount R", value=0.0)
        st.write("Line Items")
        li1,li2,li3,li4 = st.columns(4)
        item_desc=li1.text_input("Item")
        qty=li2.number_input("Qty", value=1.0)
        unit=li3.text_input("Unit", value="pcs")
        unit_price=li4.number_input("Unit Price", value=0.0)
        terms = st.text_area("Terms", value="60% deposit, 30% on delivery, 10% on completion. Valid 7 days.")
        if st.form_submit_button("Create Quote"):
            if project_options and proj_id:
                pid=project_options[proj_id]
                proj_row=fetch_one("SELECT * FROM projects WHERE id=?", (pid,))
                subtotal=qty*unit_price
                markup_val=subtotal*markup/100
                vat_val=(subtotal+markup_val-discount)*vat/100
                total=subtotal+markup_val-vat+vat_val-discount
                quote_no=f"NV-{datetime.now().year}-{int(datetime.now().timestamp())%100000}"
                q("""INSERT INTO quotes
                    (quote_no, project_id, client_name, client_phone, address, description, subtotal, markup_pct, markup_value, discount, vat_pct, vat_value, total, valid_until, terms, status, created_at)
                    VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?,?,?, ?,?)""",
                  (quote_no, pid, proj_row["client_name"], proj_row["client_phone"], proj_row["address"], desc, subtotal, markup, markup_val, discount, vat, vat_val, total, str(valid_until), terms, "Draft", datetime.now().isoformat()))
                qid=fetch_one("SELECT id FROM quotes WHERE quote_no=?", (quote_no,))["id"]
                q("INSERT INTO quote_items (quote_id, description, qty, unit, unit_price, line_total) VALUES (?,?,?,?,?,?)",
                  (qid, item_desc, qty, unit, unit_price, subtotal))
                log_action("CREATE","quote",quote_no,f"R{total:,.0f}")
                st.success(f"Quote {quote_no} created R{total:,.0f}")
                # PDF
                quote_row=fetch_one("SELECT * FROM quotes WHERE id=?", (qid,))
                items=fetch_all("SELECT * FROM quote_items WHERE quote_id=?", (qid,))
                path=quote_pdf(quote_row, items, quote_row["client_name"], quote_row["address"])
                with open(path,"rb") as f:
                    st.download_button("Download Quote PDF", f, file_name=f"{quote_no}.pdf")
                wa_msg=quote_message(proj_row["name"], proj_row["client_name"] or "Client", quote_no, total)
                st.markdown(f"[💬 Send Quote via WhatsApp]({wa_link(proj_row['client_phone'], wa_msg)})")
    conn=get_conn()
    df=pd.read_sql_query("SELECT * FROM quotes ORDER BY id DESC", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)

elif page=="🧾 Invoices":
    st.title("🧾 Invoices")
    with st.form("inv_form"):
        a,b = st.columns(2)
        proj_sel = a.selectbox("Project", list(project_options.keys()) if project_options else ["None"])
        inv_date=b.date_input("Invoice Date", date.today())
        due=b.date_input("Due Date", date.today()+timedelta(days=7))
        desc=st.text_area("Description")
        c,d = st.columns(2)
        item_desc=c.text_input("Item Description")
        amt=d.number_input("Amount R", value=0.0)
        bank=st.text_input("Bank Details", value="Nova Vital | Capitec | Acc 123456 | Ref Invoice No")
        if st.form_submit_button("Create Invoice"):
            if project_options:
                pid=project_options[proj_sel]
                proj_row=fetch_one("SELECT * FROM projects WHERE id=?", (pid,))
                inv_no=f"INV-{datetime.now().year}-{int(datetime.now().timestamp())%100000}"
                q("""INSERT INTO invoices
                    (invoice_no, project_id, client_name, invoice_date, due_date, description, subtotal, total, balance_due, status, bank_details, created_at)
                    VALUES (?,?,?,?,?,?,?,?,?,?,?,?)""",
                  (inv_no, pid, proj_row["client_name"], str(inv_date), str(due), desc, amt, amt, amt, "Not Paid", bank, datetime.now().isoformat()))
                iid=fetch_one("SELECT id FROM invoices WHERE invoice_no=?", (inv_no,))["id"]
                q("INSERT INTO invoice_items (invoice_id, description, qty, unit_price, line_total) VALUES (?,?,?,?,?)",
                  (iid, item_desc, 1, amt, amt))
                st.success(f"Invoice {inv_no} created")
    conn=get_conn()
    df=pd.read_sql_query("SELECT * FROM invoices ORDER BY id DESC", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)
    if not df.empty:
        for _, inv in df.iterrows():
            if inv["balance_due"]>0:
                proj_row=fetch_one("SELECT * FROM projects WHERE id=?", (inv["project_id"],))
                if proj_row:
                    wa_msg=invoice_message(proj_row["name"], inv["client_name"], inv["invoice_no"], inv["balance_due"])
                    st.markdown(f"[💬 Remind {inv['client_name']} for {inv['invoice_no']}]({wa_link(proj_row['client_phone'], wa_msg)})")

elif page=="🔄 Variations":
    st.title("🔄 Variation Orders")
    conn=get_conn()
    df=pd.read_sql_query("SELECT v.*, p.name as project_name, p.client_phone, p.client_name FROM variations v JOIN projects p ON v.project_id=p.id ORDER BY v.id DESC", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)
    if not df.empty:
        for _, v in df.iterrows():
            col1,col2 = st.columns(2)
            if v["status"]!="Approved":
                if col1.button(f"Approve {v['variation_no']}", key=f"app_{v['id']}"):
                    q("UPDATE variations SET status='Approved' WHERE id=?", (v["id"],))
                    st.rerun()
            path=variation_pdf(v, v["project_name"])
            with open(path,"rb") as f:
                col2.download_button(f"PDF {v['variation_no']}", f, file_name=f"{v['variation_no']}.pdf", key=f"pdf_{v['id']}")

elif page=="🛒 Procurement":
    st.title("🛒 Procurement")
    conn=get_conn()
    df=pd.read_sql_query("SELECT pr.*, p.name as project_name FROM procurement pr JOIN projects p ON pr.project_id=p.id", conn)
    conn.close()
    if not df.empty:
        not_ordered=df[df["status"]=="Required"]
        ordered=df[df["status"]=="Ordered"]
        c1,c2,c3 = st.columns(3)
        c1.metric("🔴 Not Ordered", len(not_ordered))
        c2.metric("🟡 Ordered Not Received", len(ordered))
        c3.metric("Total Value", money(df["actual_price"].sum() + df["quoted_price"].sum()))
        st.dataframe(df, use_container_width=True)

elif page=="📦 Inventory":
    st.title("📦 Workshop Inventory")
    with st.form("inv_add"):
        a,b,c,d = st.columns(4)
        item=a.text_input("Item*")
        cat=b.text_input("Category", placeholder="Board, Hardware")
        qty=c.number_input("Qty", value=0.0)
        reorder=d.number_input("Reorder Level", value=5.0)
        e,f,g = st.columns(3)
        cost=e.number_input("Unit Cost", value=0.0)
        unit=f.text_input("Unit", value="pcs")
        sup=g.text_input("Supplier")
        if st.form_submit_button("Add/Update Stock"):
            if item:
                existing=fetch_one("SELECT * FROM inventory WHERE item=?", (item,))
                if existing:
                    q("UPDATE inventory SET quantity=?, reorder_level=?, unit_cost=? WHERE item=?", (qty, reorder, cost, item))
                else:
                    q("INSERT INTO inventory (item, category, quantity, unit, reorder_level, unit_cost, supplier, created_at) VALUES (?,?,?,?,?,?,?,?)",
                      (item, cat, qty, unit, reorder, cost, sup, datetime.now().isoformat()))
                st.rerun()
    conn=get_conn()
    df=pd.read_sql_query("SELECT * FROM inventory", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)
    if not df.empty:
        low=df[df["quantity"]<=df["reorder_level"]]
        if not low.empty:
            st.error(f"⚠️ Low stock alert: {', '.join(low['item'].tolist())}")

elif page=="👷 Workers":
    st.title("👷 Workers & Wages")
    with st.form("worker_add"):
        a,b,c,d = st.columns(4)
        name=a.text_input("Name*")
        phone=b.text_input("Phone")
        role=c.text_input("Role", value="Installer")
        rate=d.number_input("Daily Rate", value=500.0)
        if st.form_submit_button("Add Worker"):
            if name:
                try:
                    q("INSERT INTO workers (name, phone, role, default_rate, created_at) VALUES (?,?,?,?,?)",
                      (name, phone, role, rate, datetime.now().isoformat()))
                    st.rerun()
                except:
                    st.error("Worker exists")
    conn=get_conn()
    workers=pd.read_sql_query("SELECT * FROM workers", conn)
    wp=pd.read_sql_query("SELECT wp.*, w.name as worker_name, p.name as project_name FROM worker_payments wp JOIN workers w ON wp.worker_id=w.id JOIN projects p ON wp.project_id=p.id", conn)
    conn.close()
    st.dataframe(workers, use_container_width=True)
    st.subheader("Pay Worker")
    if not workers.empty and project_options:
        with st.form("pay_worker"):
            a,b,c,d = st.columns(4)
            w_sel=a.selectbox("Worker", workers["name"].tolist())
            p_sel=b.selectbox("Project", list(project_options.keys()))
            p_date=c.date_input("Date", date.today())
            days=d.number_input("Days", value=1.0, step=0.5)
            rate=st.number_input("Rate R", value=float(workers[workers["name"]==w_sel]["default_rate"].iloc[0]) if not workers[workers["name"]==w_sel].empty else 500.0)
            amt=days*rate
            st.write(f"Amount: R{amt:,.2f}")
            if st.form_submit_button("Pay"):
                wid=int(workers[workers["name"]==w_sel]["id"].iloc[0])
                pid=project_options[p_sel]
                q("INSERT INTO worker_payments (project_id, worker_id, payment_date, days, rate, amount, created_at) VALUES (?,?,?,?,?,?,?)",
                  (pid, wid, str(p_date), days, rate, amt, datetime.now().isoformat()))
                st.success(f"Paid R{amt:,.2f}")
                st.rerun()
    st.dataframe(wp, use_container_width=True)

elif page=="✅ Tasks & Snags":
    st.title("✅ Tasks & Snags Overview")
    conn=get_conn()
    tasks=pd.read_sql_query("SELECT t.*, p.name as project_name FROM tasks t JOIN projects p ON t.project_id=p.id", conn)
    snags=pd.read_sql_query("SELECT s.*, p.name as project_name FROM snags s JOIN projects p ON s.project_id=p.id", conn)
    conn.close()
    c1,c2 = st.columns(2)
    c1.subheader("Tasks")
    c1.dataframe(tasks, use_container_width=True)
    c2.subheader("Snags")
    c2.dataframe(snags, use_container_width=True)

elif page=="📓 Site Diary":
    st.title("📓 Site Diary - All Projects")
    conn=get_conn()
    df=pd.read_sql_query("SELECT sl.*, p.name as project_name FROM site_logs sl JOIN projects p ON sl.project_id=p.id ORDER BY log_date DESC", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)

elif page=="📈 Leads":
    st.title("📈 Leads / Sales Pipeline")
    with st.form("lead_form"):
        a,b,c,d = st.columns(4)
        name=a.text_input("Lead / Company*")
        phone=b.text_input("Phone")
        source=c.selectbox("Source", ["Referral","Facebook","Google","Developer","Repeat Client","WhatsApp","Other"])
        ptype=d.text_input("Project Type")
        e,f = st.columns(2)
        value=e.number_input("Estimated Value", min_value=0.0)
        status=f.selectbox("Status", ["New","Contacted","Site Visit","Quoted","Negotiating","Won","Lost"])
        notes=st.text_area("Notes")
        if st.form_submit_button("Add Lead") and name:
            q("INSERT INTO leads (name, phone, source, project_type, estimated_value, status, notes, created_at) VALUES (?,?,?,?,?,?,?,?)",
              (name, phone, source, ptype, value, status, notes, datetime.now().isoformat()))
            st.rerun()
    conn=get_conn()
    df=pd.read_sql_query("SELECT * FROM leads ORDER BY id DESC", conn)
    conn.close()
    st.dataframe(df, use_container_width=True)
    if not df.empty:
        c1,c2,c3 = st.columns(3)
        c1.metric("Pipeline", money(df[df["status"].isin(["New","Contacted","Site Visit","Quoted","Negotiating"])]["estimated_value"].sum()))
        c2.metric("Won", money(df[df["status"]=="Won"]["estimated_value"].sum()))
        c3.metric("Lost", money(df[df["status"]=="Lost"]["estimated_value"].sum()))

elif page=="🤖 AI Advisor":
    st.title("🤖 AI Financial Advisor")
    if not project_options:
        st.warning("Create project first")
        st.stop()
    sel=st.selectbox("Select Project", list(project_options.keys()))
    pid=project_options[sel]
    st.markdown(f'<div style="background:linear-gradient(135deg,#151c31,#202944);padding:18px;border-radius:14px;border:1px solid #35405e">{advisor_text(pid)}</div>', unsafe_allow_html=True)
    st.divider()
    st.subheader("🔮 Scenario Calculator")
    c1,c2,c3 = st.columns(3)
    extra=c1.number_input("Extra spend R", value=0.0, step=500.0)
    extra_pay=c2.number_input("If client pays R", value=0.0, step=1000.0)
    labor_inc=c3.number_input("Labor increase %", value=0.0, step=5.0)
    res=scenario_calc(pid, extra, extra_pay, labor_inc)
    st.info(f"After scenario: Profit R{res['new_profit']:,.2f} Margin {res['new_margin']:.1f}% Cash R{res['new_cash_position']:,.2f} Outstanding R{res['new_outstanding']:,.2f}")

elif page=="📊 Reports":
    st.title("📊 Reports & BI")
    stats=company_stats()
    df=stats["df"]
    if df.empty:
        st.info("No projects")
        st.stop()
    st.dataframe(df, use_container_width=True)
    csv=df.to_csv(index=False).encode()
    st.download_button("Download Company P&L CSV", csv, "nova_vital_pl.csv")
    # Excel
    try:
        import io
        output=io.BytesIO()
        with pd.ExcelWriter(output, engine='openpyxl') as writer:
            df.to_excel(writer, sheet_name="P&L", index=False)
        st.download_button("Download Excel", output.getvalue(), "nova_vital.xlsx")
    except:
        pass
    if project_options:
        sel=st.selectbox("Detailed PDF Report", list(project_options.keys()))
        if st.button("Generate PDF"):
            pid=project_options[sel]
            proj=fetch_one("SELECT * FROM projects WHERE id=?", (pid,))
            s=project_stats(pid)
            conn=get_conn()
            exp_df=pd.read_sql_query("SELECT * FROM expenses WHERE project_id=?", conn, params=(pid,))
            conn.close()
            path=project_report_pdf(proj, s, exp_df)
            with open(path,"rb") as f:
                st.download_button("Download PDF", f, file_name=f"{sel}_Report.pdf")

elif page=="⚙️ Settings":
    st.title("⚙️ Settings")
    st.write(f"Database: {DB_FILE} | PIN via env NOVA_PIN (current default 1234)")
    st.write("Migrations: financial_db.json -> SQLite auto-migrated with backup")
    if st.button("Logout"):
        st.session_state.auth=False
        st.rerun()
    st.divider()
    st.subheader("Audit Log")
    conn=get_conn()
    logs=pd.read_sql_query("SELECT * FROM audit_log ORDER BY id DESC LIMIT 200", conn)
    conn.close()
    st.dataframe(logs, use_container_width=True)
    st.subheader("Backup")
    if os.path.exists(DB_FILE):
        with open(DB_FILE,"rb") as f:
            st.download_button("Download SQLite DB", f, file_name="nova_vital_backup.db")

st.sidebar.divider()
st.sidebar.caption("Nova Vital Advisor OS - Production v3")
