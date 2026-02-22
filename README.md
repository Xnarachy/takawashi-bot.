# takawashi-bot.
Takawashi Automobiles Ltd 
import streamlit as st
import openai
import matplotlib.pyplot as plt
import pandas as pd
import io
import re

# -----------------------------------------------------------------------------
# 1. PAGE CONFIGURATION & BRANDING
# -----------------------------------------------------------------------------
st.set_page_config(
    page_title="Takawashi Ops Assistant",
    page_icon="🚗",
    layout="wide"
)

# Custom CSS for Takawashi Branding (Dark Blue & Red)
st.markdown("""
<style>
    .main-header {font-size: 2.5rem; font-weight: 800; color: #2c3e50; margin-bottom: 0.5rem;}
    .sub-header {font-size: 1.1rem; color: #7f8c8d; margin-bottom: 2rem;}
    .stButton>button {
        background-color: #c0392b; 
        color: white; 
        font-weight: bold; 
        border-radius: 4px;
        border: none;
    }
    .stButton>button:hover {background-color: #a93226;}
    div[data-testid="stMarkdownContainer"] p {font-size: 1.05rem;}
</style>
""", unsafe_allow_html=True)

st.markdown('<p class="main-header">🚗 Takawashi Automobiles Ltd</p>', unsafe_allow_html=True)
st.markdown('<p class="sub-header">Functional Operations Assistant | Email Rewriting • Follow-ups • Data Visualization</p>', unsafe_allow_html=True)

# -----------------------------------------------------------------------------
# 2. SIDEBAR & API CONFIG
# -----------------------------------------------------------------------------
with st.sidebar:
    st.header("⚙️ Configuration")
    api_key = st.text_input("OpenAI API Key", type="password", placeholder="sk-...")
    
    st.divider()
    st.markdown("### 📋 How to Use")
    st.markdown("**1. Email Tasks:** Paste messy mechanic notes or customer queries to rewrite them professionally.")
    st.markdown("**2. Follow-ups:** Request specific follow-up messages (e.g., '3-day post-service check').")
    st.markdown("**3. Charts:** Paste data (CSV format) and ask for a specific chart (e.g., 'Bar chart of monthly sales').")
    
    if not api_key:        st.warning("⚠️ Please enter your API Key to activate the bot.")
        st.stop()

# Initialize OpenAI Client
client = openai.OpenAI(api_key=api_key)

# -----------------------------------------------------------------------------
# 3. SYSTEM PROMPT (The "Brain")
# -----------------------------------------------------------------------------
SYSTEM_INSTRUCTION = """
You are the Executive Operations Assistant for **Takawashi Automobiles Ltd**.
Your purpose is strictly functional: to generate ready-to-use business assets (emails, messages, charts).

### RULES FOR TEXT TASKS (Emails/Messages):
1. **Tone:** Professional, direct, trustworthy, and safety-conscious. No slang.
2. **Format:** 
   - Always include a clear **Subject Line**.
   - Use bullet points for costs, service items, or action steps.
   - End with a specific Call to Action (e.g., "Reply to approve," "Call to schedule").
3. **Constraint:** NEVER use filler phrases like "I hope this email finds you well," "Just checking in," or "Let me know if you have any questions." Start directly with the value proposition or issue.
4. **Context:** If rewriting mechanic notes, translate technical jargon into clear customer language while maintaining accuracy.

### RULES FOR VISUAL TASKS (Charts):
1. If the user provides data (CSV, list, or table) and requests a visualization:
   - You MUST write valid Python code using `matplotlib` and `pandas`.
   - **Style:** Use Takawashi brand colors: Dark Blue (#2c3e50), Red (#c0392b), and Grey (#95a5a6).
   - **Execution:** The code MUST save the plot to a variable named `buf` (BytesIO) using `plt.savefig(buf, format='png')`. 
   - **Constraint:** Do NOT use `plt.show()`.
   - Ensure titles and axis labels are professional and clear.
2. Wrap the code strictly in markdown blocks: ```python ... ```.

### GENERAL CONSTRAINTS:
- Output ONLY the requested asset. Do not add conversational intros like "Here is your email" or "I have generated the chart."
- If data is insufficient for a chart, ask exactly what numbers are missing.
"""

# -----------------------------------------------------------------------------
# 4. SESSION STATE MANAGEMENT
# -----------------------------------------------------------------------------
if "messages" not in st.session_state:
    st.session_state.messages = [{"role": "system", "content": SYSTEM_INSTRUCTION}]

# Display Chat History
for message in st.session_state.messages:
    if message["role"] == "system":
        continue
    
    with st.chat_message(message["role"]):
        # Display Text Content
        if "content" in message and message["content"]:            st.markdown(message["content"])
        
        # Display Image Content (if present from previous turn)
        if "image_data" in message:
            st.image(message["image_data"], caption="Takawashi Data Report", use_container_width=True)
            
            # Download Button for the Chart
            buf = message["image_buffer"]
            buf.seek(0)
            st.download_button(
                label="📥 Download Chart (PNG)",
                data=buf.getvalue(),
                file_name="takawashi_report.png",
                mime="image/png"
            )

# -----------------------------------------------------------------------------
# 5. MAIN INTERACTION LOOP
# -----------------------------------------------------------------------------
prompt = st.chat_input("Paste notes, request a follow-up, or provide data for a chart...")

if prompt:
    # Add User Message to History
    st.session_state.messages.append({"role": "user", "content": prompt})
    with st.chat_message("user"):
        st.markdown(prompt)

    # Generate Response
    with st.chat_message("assistant"):
        with st.spinner("Processing task..."):
            try:
                response = client.chat.completions.create(
                    model="gpt-4o",
                    messages=st.session_state.messages,
                    temperature=0.3, # Low temperature for consistency
                )
                
                assistant_content = response.choices[0].message.content
                
                # Check if the response contains Python code for a chart
                code_match = re.search(r"```python(.*?)```", assistant_content, re.DOTALL)
                
                if code_match and ("matplotlib" in assistant_content or "plt." in assistant_content):
                    st.markdown("📊 **Generating Visualization...**")
                    code_to_run = code_match.group(1)
                    
                    # Prepare buffer for image
                    img_buffer = io.BytesIO()
                    
                    # Define safe execution environment                    local_vars = {
                        "pd": pd, 
                        "plt": plt, 
                        "io": io, 
                        "buf": img_buffer
                    }
                    
                    try:
                        # Execute the generated code
                        exec(code_to_run, {}, local_vars)
                        
                        # Retrieve image
                        img_buffer.seek(0)
                        
                        # Display Image
                        st.image(img_buffer, caption="Generated Report", use_container_width=True)
                        
                        # Download Button
                        st.download_button(
                            label="📥 Download Chart (PNG)",
                            data=img_buffer.getvalue(),
                            file_name="takawashi_report.png",
                            mime="image/png"
                        )
                        
                        # Save to history (store both text confirmation and image)
                        st.session_state.messages.append({
                            "role": "assistant",
                            "content": "Chart generated successfully based on provided data.",
                            "image_data": img_buffer,
                            "image_buffer": img_buffer
                        })
                        
                    except Exception as e:
                        st.error(f"❌ **Chart Generation Failed:** {str(e)}")
                        st.code(code_to_run)
                        st.session_state.messages.append({"role": "assistant", "content": f"Error generating chart: {str(e)}"})
                
                else:
                    # Standard Text Response
                    st.markdown(assistant_content)
                    st.session_state.messages.append({"role": "assistant", "content": assistant_content})

            except Exception as e:
                st.error(f"System Error: {str(e)}")
