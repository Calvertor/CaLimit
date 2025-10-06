from pywebio import start_server
from pywebio.input import input, input_group, TEXT, select, textarea
from pywebio.output import put_html, clear, put_button, put_collapse, put_table, put_loading, toast
import sympy as sp
import numpy as np
import plotly.graph_objects as go
import plotly.io as pio
import re
from fractions import Fraction
import math
import time
from functools import lru_cache

# ---------- Enhanced Color System ----------
COLORS = {
    'bg': '#FFF5E1', 
    'accent': '#D7A58F', 
    'secondary': '#A67C52',
    'text': '#5C3A21', 
    'plot_bg': '#FDF6F0',
    'button': '#8B5A2B',
    'button_hover': '#6D4424',
    'card': '#FAEBD7',
    'error': '#FF6B6B',
    'success': '#4ECDC4',
    'warning': '#FFE66D',
    'left': '#4ECDC4',
    'right': '#FF6B6B',
    'both': '#45B7D1',
    'theory': '#6A89CC',
    'trig': '#FF9FF3',
    'infinity': '#706FD3',
    'threed': '#FF9F1C',
    'step1': '#FF9F1C',
    'step2': '#4ECDC4',
    'step3': '#FF6B6B',
    'step4': '#6A89CC'
}

FONT = "'Nunito', sans-serif"

# ---------- Security & Sanitization ----------
def sanitize_expression(expr):
    """Enhanced security sanitization"""
    if not expr:
        return ""
    
    # Remove dangerous patterns
    dangerous_patterns = [
        r'__.*__', r'import\s+\w+', r'exec\s*\(', r'eval\s*\(',
        r'open\s*\(', r'file\s*\(', r'os\.', r'sys\.', r'subprocess\.',
        r'breakpoint', r'compile', r'globals', r'locals'
    ]
    
    for pattern in dangerous_patterns:
        expr = re.sub(pattern, '[REMOVED]', expr, flags=re.IGNORECASE)
    
    # Limit length
    if len(expr) > 500:
        raise ValueError("❌ Ekspresi terlalu panjang (max 500 karakter)")
    
    return expr.strip()

# ---------- Enhanced Expression Parser ----------
def parse_expr(expr):
    """Cached expression parser dengan handling lebih robust"""
    if not expr or not expr.strip():
        return ""
    
    try:
        expr = sanitize_expression(expr)
        original_expr = expr
        
        # Enhanced trigonometric handling
        trig_patterns = {
            'sin': 'sin', 'cos': 'cos', 'tan': 'tan', 
            'cot': 'cot', 'sec': 'sec', 'csc': 'csc',
            'arcsin': 'asin', 'arccos': 'acos', 'arctan': 'atan'
        }
        
        for natural, sympy_func in trig_patterns.items():
            expr = re.sub(rf'{natural}\s*\(\s*([^)]+)\s*\)', rf'{sympy_func}(\1)', expr, flags=re.IGNORECASE)
        
        # Enhanced power handling
        expr = expr.replace('²', '**2').replace('³', '**3')
        expr = re.sub(r'(\d+|[a-zA-Z])\^(\d+|[a-zA-Z])', r'\1**\2', expr)
        
        # Enhanced implicit multiplication
        expr = re.sub(r'(\d+)([a-zA-Z\(])', r'\1*\2', expr)
        expr = re.sub(r'([a-zA-Z\)])(\d+)', r'\1*\2', expr)
        expr = re.sub(r'([a-zA-Z\)])\s*\(', r'\1*(', expr)
        expr = re.sub(r'(\))\s*([a-zA-Z\d])', r'\1*\2', expr)
        
        # Enhanced special functions
        expr = expr.replace('ln', 'log')
        expr = expr.replace('π', 'pi').replace('Pi', 'pi')
        expr = expr.replace('∞', 'oo').replace('infinity', 'oo')
        expr = expr.replace('e^', 'exp').replace('exp(', 'exp(')
        
        # Handle square root and other roots
        expr = re.sub(r'√\s*\(([^)]+)\)', r'sqrt(\1)', expr, flags=re.IGNORECASE)
        expr = re.sub(r'sqrt\s*\(([^)]+)\)', r'sqrt(\1)', expr, flags=re.IGNORECASE)
        
        # Final cleanup
        expr = re.sub(r'\s+', '', expr)  # Remove all whitespace
        
        # Validate the expression
        x = sp.symbols('x')
        test_expr = sp.sympify(expr)
        
        return expr
        
    except Exception as e:
        raise ValueError(f"❌ Tidak bisa parse ekspresi: {str(e)}")

# ---------- Enhanced Display Formatter ----------
def format_display_expr(expr_str):
    """Format ekspresi untuk tampilan yang natural dan beautiful"""
    if not expr_str:
        return ""
    
    expr_str = str(expr_str)
    
    # Convert back to natural format
    replacements = [
        (r'\*\*', '^'),
        (r'sqrt\(([^)]+)\)', r'√(\1)'),
        (r'pi', 'π'),
        (r'oo', '∞'),
        (r'exp\(([^)]+)\)', r'e^(\1)'),
        (r'asin', 'arcsin'),
        (r'acos', 'arccos'),
        (r'atan', 'arctan'),
        (r'\^2', '²'),
        (r'\^3', '³'),
        (r'(\d)\*', r'\1'),  # Remove * after numbers
        (r'\)\*\(', ')('),
    ]
    
    for pattern, replacement in replacements:
        expr_str = re.sub(pattern, replacement, expr_str)
    
    return expr_str

# ---------- Format Limit Value ----------
def format_limit_value(value):
    """Format nilai limit menjadi lebih rapi"""
    if value in ['oo', '-oo', 'zoo', 'nan', 'Tidak dapat dihitung']:
        return '∞' if value == 'oo' else '-∞' if value == '-oo' else 'Tidak terdefinisi'
    
    if isinstance(value, (int, float, sp.Integer, sp.Float)):
        try:
            float_val = float(value)
            # Pastikan integer ditampilkan sebagai integer
            if abs(float_val - round(float_val)) < 1e-10:
                return str(int(round(float_val)))
            
            # Cek jika pecahan sederhana
            try:
                frac = Fraction(float_val).limit_denominator(1000)
                if frac.denominator != 1 and abs(frac.numerator/frac.denominator - float_val) < 1e-10:
                    return f"{frac.numerator}/{frac.denominator}"
            except:
                pass
            
            # Format dengan presisi wajar, hilangkan .0 jika integer
            if float_val == int(float_val):
                return str(int(float_val))
            
            formatted = f"{float_val:.6f}"
            if '.' in formatted:
                formatted = formatted.rstrip('0').rstrip('.')
            return formatted
        except:
            return str(value)
    
    # Handle sympy objects
    if hasattr(value, 'evalf'):
        try:
            float_val = float(value.evalf())
            if float_val == int(float_val):
                return str(int(float_val))
            return f"{float_val:.6f}".rstrip('0').rstrip('.')
        except:
            pass
    
    return str(value)

# ---------- Validate Expression ----------
def validate_expression(expr):
    """Check if expression is valid"""
    if not expr or not expr.strip():
        return False, "Ekspresi tidak boleh kosong"
    
    try:
        x = sp.symbols('x')
        parsed_expr = parse_expr(expr)
        sp.sympify(parsed_expr)
        return True, "Ekspresi valid"
    except Exception as e:
        return False, f"Ekspresi tidak valid: {str(e)}"

# ---------- Parse X Value ----------
def parse_x_value(x_str):
    """Parse nilai x dengan support untuk 0+, 1-, dll."""
    if not x_str:
        return None, "Nilai x tidak boleh kosong"
    
    x_str = x_str.strip()
    
    # Handle infinity cases
    if x_str.lower() in ['inf', 'infinity', '∞']:
        return 'oo', None
    elif x_str.lower() in ['-inf', '-infinity', '-∞']:
        return '-oo', None
    
    # Handle cases like 0+, 0-, 1+, 1-
    if x_str.endswith('+') or x_str.endswith('-'):
        try:
            base_val = float(x_str[:-1])
            direction = 'right' if x_str.endswith('+') else 'left'
            return base_val, direction
        except:
            return None, "Format nilai x tidak valid"
    
    # Handle regular numbers
    try:
        return float(x_str), None
    except:
        return None, "Nilai x harus berupa angka"

# ---------- Calculate Limit ----------
def calc_limit(expr_str, x_val, direction='both'):
    x = sp.symbols('x')
    try:
        parsed_expr = parse_expr(expr_str)
        expr = sp.sympify(parsed_expr)
        
        if x_val == 'oo':
            x_val = sp.oo
        elif x_val == '-oo':
            x_val = -sp.oo
        
        if direction == 'both': 
            result = sp.limit(expr, x, x_val)
        elif direction == 'left': 
            result = sp.limit(expr, x, x_val, dir='-')
        else: 
            result = sp.limit(expr, x, x_val, dir='+')
        
        return format_limit_value(result)
    except Exception as e:
        return "Tidak dapat dihitung"

# ---------- Smart Limit Calculator ----------
def smart_limit_calculation(expr_str, x_val, direction='both'):
    """Enhanced limit calculation dengan step-by-step analysis"""
    x = sp.symbols('x')
    steps = []
    
    try:
        parsed_expr = parse_expr(expr_str)
        expr = sp.sympify(parsed_expr)
        
        steps.append(f"🎯 <b>Step 1: Analisis Fungsi</b>")
        steps.append(f"   f(x) = {format_display_expr(expr_str)}")
        steps.append(f"   x → {x_val}{'⁻' if direction == 'left' else '⁺' if direction == 'right' else ''}")
        
        # Step 2: Simplifikasi
        simplified = sp.simplify(expr)
        if simplified != expr:
            steps.append(f"🔧 <b>Step 2: Simplifikasi</b>")
            steps.append(f"   f(x) = {format_display_expr(str(simplified))}")
            expr = simplified
        
        # Step 3: Substitusi langsung
        steps.append(f"🔍 <b>Step 3: Substitusi Langsung</b>")
        try:
            if x_val in ['oo', '-oo']:
                direct_sub = "Tidak bisa substitusi langsung di tak hingga"
            else:
                direct_val = expr.subs(x, x_val)
                steps.append(f"   f({x_val}) = {format_limit_value(direct_val)}")
        except:
            steps.append(f"   f({x_val}) = Tidak terdefinisi (bentuk tak tentu)")
        
        # Step 4: Hitung limit
        steps.append(f"📊 <b>Step 4: Hitung Limit</b>")
        
        if x_val == 'oo':
            x_val_sym = sp.oo
        elif x_val == '-oo':
            x_val_sym = -sp.oo
        else:
            x_val_sym = x_val
        
        if direction == 'both':
            result = sp.limit(expr, x, x_val_sym)
        elif direction == 'left':
            result = sp.limit(expr, x, x_val_sym, dir='-')
        else:
            result = sp.limit(expr, x, x_val_sym, dir='+')
        
        steps.append(f"   Hasil limit = <b>{format_limit_value(result)}</b>")
        
        return format_limit_value(result), steps
        
    except Exception as e:
        return "Tidak dapat dihitung", [f"❌ <b>Error:</b> {str(e)}"]

# ---------- Enhanced Continuity Analysis ----------
def enhanced_continuity_analysis(expr_str, x_val):
    """Analisis kontinuitas yang lebih detail"""
    x = sp.symbols('x')
    analysis = {
        'limit_left': None,
        'limit_right': None,
        'limit_both': None,
        'func_value': None,
        'defined': False,
        'continuous': False,
        'discontinuity_type': None,
        'steps': []
    }
    
    try:
        parsed_expr = parse_expr(expr_str)
        expr = sp.sympify(parsed_expr)
        
        analysis['steps'].append("🔍 <b>Analisis Kontinuitas:</b>")
        
        # 1. Check if function is defined at point
        if x_val not in ['oo', '-oo']:
            try:
                func_val = expr.subs(x, x_val)
                analysis['func_value'] = format_limit_value(func_val)
                analysis['defined'] = True
                analysis['steps'].append(f"✅ f({x_val}) = {analysis['func_value']} (terdefinisi)")
            except:
                analysis['func_value'] = "Tidak terdefinisi"
                analysis['defined'] = False
                analysis['steps'].append(f"❌ f({x_val}) tidak terdefinisi")
        
        # 2. Calculate limits
        analysis['limit_left'] = calc_limit(expr_str, x_val, 'left')
        analysis['limit_right'] = calc_limit(expr_str, x_val, 'right')
        analysis['limit_both'] = calc_limit(expr_str, x_val, 'both')
        
        analysis['steps'].append(f"📊 Limit kiri: {analysis['limit_left']}")
        analysis['steps'].append(f"📊 Limit kanan: {analysis['limit_right']}")
        
        # 3. Determine continuity and discontinuity type
        if x_val in ['oo', '-oo']:
            analysis['continuous'] = False
            analysis['discontinuity_type'] = "Titik tak hingga"
        elif not analysis['defined']:
            analysis['continuous'] = False
            if analysis['limit_left'] == analysis['limit_right'] != "Tidak dapat dihitung":
                analysis['discontinuity_type'] = "Diskontinuitas yang dapat dihapus"
            else:
                analysis['discontinuity_type'] = "Diskontinuitas esensial"
        else:
            if (analysis['limit_left'] == analysis['limit_right'] == analysis['limit_both'] == 
                analysis['func_value'] != "Tidak dapat dihitung"):
                analysis['continuous'] = True
                analysis['discontinuity_type'] = "Kontinu"
            else:
                analysis['continuous'] = False
                if analysis['limit_left'] != analysis['limit_right']:
                    analysis['discontinuity_type'] = "Diskontinuitas lompatan"
                else:
                    analysis['discontinuity_type'] = "Diskontinuitas yang dapat dihapus"
        
        analysis['steps'].append(f"🎯 <b>Kesimpulan:</b> {analysis['discontinuity_type']}")
        
    except Exception as e:
        analysis['error'] = str(e)
        analysis['steps'].append(f"❌ Error dalam analisis: {str(e)}")
    
    return analysis

# ---------- Interactive Examples System ----------
INTERACTIVE_EXAMPLES = [
    {
        "name": "🔺 Limit Trigonometri Klasik",
        "expr": "sin(x)/x",
        "xval": "0",
        "description": "Limit fundamental sin(x)/x ketika x→0"
    },
    {
        "name": "∞ Limit di Tak Hingga",
        "expr": "(2x^2 + 3)/(x^2 - 1)", 
        "xval": "infinity",
        "description": "Perilaku fungsi rasional ketika x→∞"
    },
    {
        "name": "√ Limit Fungsi Akar",
        "expr": "sqrt(x-2)",
        "xval": "2+",
        "description": "Limit satu sisi untuk fungsi akar"
    },
    {
        "name": "📈 Limit Eksponensial",
        "expr": "(1 + 1/x)^x",
        "xval": "infinity", 
        "description": "Limit penting menuju bilangan e"
    },
    {
        "name": "🎭 Limit Piecewise",
        "expr": "(x^2 - 1)/(x - 1)",
        "xval": "1",
        "description": "Bentuk tak tentu 0/0 yang dapat disederhanakan"
    }
]

def create_example_buttons():
    """Create interactive example buttons"""
    examples_js = []
    for i, example in enumerate(INTERACTIVE_EXAMPLES):
        examples_js.append({
            "expr": example["expr"],
            "xval": example["xval"],
            "name": example["name"]
        })
    
    html = f"""
    <div style='padding:20px; background:#6A89CC20; border-radius:15px; margin-bottom:20px;'>
        <h3 style='color:#6A89CC; font-family:{FONT}; margin-top:0;'>🚀 Contoh Cepat</h3>
        <p style='color:#5C3A21;'>Klik contoh untuk langsung mencoba:</p>
        <div style='display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 10px; margin-top: 15px;'>
    """
    
    for i, example in enumerate(INTERACTIVE_EXAMPLES):
        html += f"""
            <div style='background:white; padding:15px; border-radius:10px; box-shadow:0 2px 8px rgba(0,0,0,0.1); cursor:pointer; transition:all 0.3s;'
                 onclick='loadExample({i})'
                 onmouseover='this.style.transform="scale(1.02)"'
                 onmouseout='this.style.transform="scale(1)"'>
                <h4 style='color:#6A89CC; margin:0 0 8px 0; font-size:14px;'>{example["name"]}</h4>
                <p style='margin:5px 0; font-size:12px; color:#666;'>{example["description"]}</p>
                <p style='margin:5px 0; font-size:11px; color:#888;'>f(x) = {format_display_expr(example["expr"])}, x→{example["xval"]}</p>
            </div>
        """
    
    html += f"""
        </div>
    </div>
    <script>
    const examples = {examples_js};
    
    function loadExample(index) {{
        const example = examples[index];
        document.querySelector('input[name="expr1"]').value = example.expr;
        document.querySelector('input[name="xval"]').value = example.xval;
        document.querySelector('select[name="direction"]').value = 'both';
        
        // Show success message
        const toast = document.createElement('div');
        toast.style.cssText = 'position:fixed; top:20px; right:20px; background:#4ECDC4; color:white; padding:15px; border-radius:10px; z-index:1000;';
        toast.innerHTML = `✅ Loaded: ${{example.name}}`;
        document.body.appendChild(toast);
        setTimeout(() => toast.remove(), 3000);
    }}
    </script>
    """
    return html

# ---------- Enhanced Visualization ----------
def create_advanced_plot(expr_str1, expr_str2, x_val, direction):
    """Enhanced plotting dengan analisis limit visual"""
    if x_val in ['oo', '-oo', '∞', '-∞']:
        return """
        <div style='padding:20px; background:#FFE66D20; border-radius:10px; text-align:center;'>
            <h4 style='color:#5C3A21;'>🌌 Visualisasi Tak Hingga</h4>
            <p>Untuk limit di tak hingga, perhatikan perilaku asimtotik fungsi ketika x membesar/mengecil</p>
        </div>
        """
    
    x = sp.symbols('x')
    try:
        x_val_float = float(x_val)
        
        # Dynamic range based on function behavior
        left_range = max(x_val_float - 3, x_val_float - 0.1) if x_val_float > 0 else x_val_float - 3
        right_range = min(x_val_float + 3, x_val_float + 0.1) if x_val_float < 0 else x_val_float + 3
        
        X = np.linspace(left_range, right_range, 1000)
        fig = go.Figure()
        
        colors = [COLORS['accent'], COLORS['secondary']]
        expressions = [(expr_str1, 'f(x)'), (expr_str2, 'g(x)')]
        
        for i, (expr_str, label) in enumerate(expressions):
            if expr_str and expr_str.strip():
                try:
                    parsed_expr = parse_expr(expr_str)
                    expr = sp.sympify(parsed_expr)
                    f = sp.lambdify(x, expr, modules=['numpy'])
                    
                    Y = []
                    for v in X:
                        try:
                            y_val = f(v)
                            Y.append(y_val if np.isfinite(y_val) else np.nan)
                        except:
                            Y.append(np.nan)
                    
                    Y = np.array(Y)
                    
                    fig.add_trace(go.Scatter(
                        x=X, y=Y, 
                        mode='lines', 
                        line=dict(color=colors[i], width=3),
                        name=f'{label} = {format_display_expr(expr_str)}',
                        hovertemplate=f'{label}: %{{y}}<extra></extra>'
                    ))
                    
                    # Add limit point if exists
                    limit_val = calc_limit(expr_str, x_val, direction)
                    if limit_val not in ['Tidak dapat dihitung', '∞', '-∞']:
                        try:
                            fig.add_trace(go.Scatter(
                                x=[x_val_float], y=[float(limit_val)],
                                mode='markers+text',
                                marker=dict(size=12, color=colors[i], symbol='diamond'),
                                text=[f'Limit: {limit_val}'],
                                textposition='top center',
                                name=f'Limit {label}',
                                showlegend=False
                            ))
                        except:
                            pass
                            
                except Exception as e:
                    continue
        
        # Add approach visualization based on direction
        if direction in ['left', 'both']:
            fig.add_vline(x=x_val_float - 0.001, line_dash="dot", line_color=COLORS['left'], opacity=0.5)
        if direction in ['right', 'both']:
            fig.add_vline(x=x_val_float + 0.001, line_dash="dot", line_color=COLORS['right'], opacity=0.5)
        
        fig.add_vline(x=x_val_float, line_dash="dash", line_color="gray", opacity=0.7)
        
        fig.update_layout(
            plot_bgcolor=COLORS['plot_bg'],
            paper_bgcolor=COLORS['plot_bg'],
            font=dict(color=COLORS['text'], family=FONT),
            title="📈 Visualisasi Limit & Pendekatan Fungsi",
            title_x=0.5,
            height=500,
            showlegend=True,
            hovermode='x unified'
        )
        
        return pio.to_html(fig, full_html=False)
        
    except Exception as e:
        return f"""
        <div style='padding:20px; background:#FFE4E4; border-radius:10px; text-align:center;'>
            <h4 style='color:{COLORS["error"]};'>❌ Error Visualisasi</h4>
            <p>Tidak dapat membuat grafik: {str(e)}</p>
        </div>
        """

# ---------- Show Usage Guide ----------
def show_usage_guide():
    """Menampilkan panduan penggunaan"""
    return f"""
    <div style='padding:20px; background:{COLORS['bg']}; border-radius:15px; margin-bottom:20px; border:2px solid {COLORS['accent']};'>
        <h3 style='color:{COLORS['accent']}; font-family:{FONT}; margin-top:0;'>📖 Panduan Penggunaan CaLimit Pro</h3>
        
        <div style='display:grid; grid-template-columns:1fr 1fr; gap:20px; margin-top:15px;'>
            <div style='background:white; padding:15px; border-radius:10px;'>
                <h4 style='color:{COLORS['accent']}; margin:0 0 10px 0;'>🎯 Format Input</h4>
                <ul style='color:{COLORS['text']}; font-size:14px; padding-left:20px;'>
                    <li><b>sin(x), cos(2x), tan(x)</b></li>
                    <li><b>√(x), ln(x), e^x</b></li>
                    <li><b>(x^2-1)/(x-1)</b></li>
                    <li><b>0, 1+, 2-, infinity</b></li>
                </ul>
            </div>
            
            <div style='background:white; padding:15px; border-radius:10px;'>
                <h4 style='color:{COLORS['theory']}; margin:0 0 10px 0;'>💡 Tips & Trik</h4>
                <ul style='color:{COLORS['text']}; font-size:14px; padding-left:20px;'>
                    <li>Gunakan <b>0+</b> untuk limit kanan</li>
                    <li>Gunakan <b>0-</b> untuk limit kiri</li>
                    <li><b>infinity</b> untuk tak hingga</li>
                    <li>Klik contoh untuk load cepat</li>
                </ul>
            </div>
        </div>
    </div>
    """

# ---------- Main Application ----------
def main():
    clear()
    
    # Enhanced Header dengan animasi
    put_html(f"""
    <div style='text-align:center; padding:30px; background:linear-gradient(135deg, {COLORS['accent']}, {COLORS['theory']}); border-radius:0 0 25px 25px; margin-bottom:30px;'>
        <h1 style='color:white; font-family:{FONT}; margin:0; font-size:2.5em; text-shadow:2px 2px 4px rgba(0,0,0,0.3);'>🧮 CaLimit Pro</h1>
        <p style='color:white; font-family:{FONT}; margin:10px 0 0 0; font-size:1.1em;'>Kalkulator Limit Intelligent dengan Analisis Lengkap</p>
        <div style='display:flex; justify-content:center; gap:10px; margin-top:15px; flex-wrap:wrap;'>
            <span style='background:rgba(255,255,255,0.3); padding:5px 15px; border-radius:20px; color:white; font-size:0.9em;'>Step-by-Step</span>
            <span style='background:rgba(255,255,255,0.3); padding:5px 15px; border-radius:20px; color:white; font-size:0.9em;'>Visualisasi 2D/3D</span>
            <span style='background:rgba(255,255,255,0.3); padding:5px 15px; border-radius:20px; color:white; font-size:0.9em;'>Analisis Kontinuitas</span>
        </div>
    </div>
    """)
    
    # Show usage guide
    put_html(show_usage_guide())
    
    # Interactive Examples
    put_html(create_example_buttons())
    
    # Enhanced Input Section
    put_html(f"""
    <div style='background:{COLORS['card']}; padding:25px; border-radius:20px; margin-bottom:25px; box-shadow:0 4px 15px rgba(0,0,0,0.1);'>
        <h3 style='color:{COLORS['text']}; font-family:{FONT}; margin-top:0; display:flex; align-items:center; gap:10px;'>
            <span style='background:{COLORS['accent']}; color:white; padding:8px 12px; border-radius:10px;'>🧩</span>
            Input Fungsi & Parameter
        </h3>
    </div>
    """)
    
    # Input form - FIXED: removed put_loading with invalid color
    data = input_group("📋 Masukkan Detail Perhitungan", [
        textarea("Fungsi pertama f(x) 🎯", name="expr1", type=TEXT, 
                placeholder="Contoh: sin(x)/x, √(x-2), (x²-1)/(x-1), ln(x+1)", 
                required=True,
                rows=2),
        textarea("Fungsi kedua g(x) (opsional)", name="expr2", type=TEXT,
                placeholder="Kosongkan jika hanya satu fungsi\nContoh: cos(x), e^x, x^2",
                rows=2),
        input("Nilai x mendekati 🎯", name="xval", type=TEXT, 
             placeholder="Contoh: 0, 1, 0+, 2-, infinity, π/2",
             required=True),
        select("Arah pendekatan:", name="direction", options=[
            {'label': '🔄 Kedua arah (default)', 'value': 'both'},
            {'label': '⬅️ Dari kiri saja (x→a⁻)', 'value': 'left'},
            {'label': '➡️ Dari kanan saja (x→a⁺)', 'value': 'right'}
        ], value='both')
    ])
    
    expr1 = data['expr1']
    expr2 = data.get('expr2', "")
    xval_str = data['xval']
    dir_choice = data['direction']
    
    # Enhanced input validation dengan user feedback
    try:
        xval, auto_direction = parse_x_value(xval_str)
        if xval is None:
            put_html(f"<div style='color:red; padding:10px; background:#FFE4E4; border-radius:5px;'>⚠️ {auto_direction}</div>")
            put_button("🔄 Coba Lagi", onclick=lambda: main(), color='danger')
            return
        
        if auto_direction:
            dir_choice = auto_direction
            put_html(f"<div style='color:green; padding:10px; background:#E8F5E8; border-radius:5px;'>🎯 Auto-detected: limit {auto_direction}</div>")
            
    except Exception as e:
        put_html(f"<div style='color:red; padding:10px; background:#FFE4E4; border-radius:5px;'>❌ Error: {str(e)}</div>")
        put_button("🔄 Coba Lagi", onclick=lambda: main(), color='danger')
        return
    
    # Validasi fungsi pertama
    is_valid, msg = validate_expression(expr1)
    if not is_valid:
        put_html(f"<div style='color:red; padding:10px; background:#FFE4E4; border-radius:5px;'>❌ {msg}</div>")
        put_button("🔄 Coba Lagi", onclick=lambda: main(), color='danger')
        return
    
    # Show loading dengan warna yang valid
    put_html("<div style='text-align:center; padding:20px;'><h3>🔄 Menghitung...</h3></div>")
    
    # Calculate results dengan enhanced display
    put_html(f"""
    <div style='background:linear-gradient(135deg, {COLORS['success']}20, {COLORS['theory']}20); padding:25px; border-radius:20px; margin-bottom:25px;'>
        <h2 style='color:{COLORS['text']}; font-family:{FONT}; margin-top:0; text-align:center;'>
            📊 HASIL PERHITUNGAN LIMIT
        </h2>
    </div>
    """)
    
    # Enhanced results display
    functions = [(expr1, "Fungsi 1")]
    if expr2 and expr2.strip():
        functions.append((expr2, "Fungsi 2"))
    
    for i, (expr, label) in enumerate(functions):
        if not expr.strip():
            continue
            
        color = list(COLORS.values())[i + 5] if i + 5 < len(COLORS) else COLORS['accent']
        
        put_html(f"""
        <div style='background:{COLORS["card"]}; padding:25px; border-radius:15px; margin-bottom:20px; border-left:5px solid {color};'>
            <h3 style='color:{color}; font-family:{FONT}; margin-top:0; display:flex; align-items:center; gap:10px;'>
                <span style='background:{color}; color:white; padding:5px 10px; border-radius:8px;'>{(i+1)}</span>
                {label}: f(x) = {format_display_expr(expr)}
            </h3>
        """)
        
        # Smart calculation dengan steps
        limit_result, steps = smart_limit_calculation(expr, xval, dir_choice)
        
        # Display steps
        put_html("<div style='background:white; padding:15px; border-radius:10px; margin:15px 0;'>")
        for step in steps:
            put_html(f"<p style='margin:8px 0; font-family:{FONT};'>{step}</p>")
        put_html("</div>")
        
        # Enhanced continuity analysis
        continuity = enhanced_continuity_analysis(expr, xval)
        if continuity['steps']:
            put_html("<div style='background:#4ECDC420; padding:15px; border-radius:10px; margin:15px 0;'>")
            for step in continuity['steps']:
                put_html(f"<p style='margin:5px 0; font-family:{FONT}; font-size:14px;'>{step}</p>")
            put_html("</div>")
        
        put_html("</div>")
    
    # Enhanced Visualization
    put_html(f"""
    <div style='background:{COLORS["card"]}; padding:25px; border-radius:15px; margin-bottom:20px;'>
        <h3 style='color:{COLORS["threed"]}; font-family:{FONT}; margin-top:0; display:flex; align-items:center; gap:10px;'>
            <span style='background:{COLORS["threed"]}; color:white; padding:5px 10px; border-radius:8px;'>📈</span>
            Visualisasi Interaktif
        </h3>
    """)
    
    plot_html = create_advanced_plot(expr1, expr2, xval, dir_choice)
    put_html(plot_html)
    put_html("</div>")
    
    # Success message
    put_html(f"""
    <div style='background:#4ECDC420; padding:15px; border-radius:10px; margin:20px 0; text-align:center;'>
        <h4 style='color:#4ECDC4; margin:0;'>✅ Perhitungan Selesai!</h4>
    </div>
    """)
    
    # Enhanced action buttons
    put_html(f"""
    <div style='display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; margin: 20px 0;'>
        <button onclick='location.reload()' style='
            background: {COLORS['success']}; 
            color: white; 
            border: none; 
            padding: 15px; 
            border-radius: 10px; 
            font-family: {FONT}; 
            font-size: 16px; 
            cursor: pointer;
            transition: all 0.3s;
        ' onmouseover='this.style.background="{COLORS['button_hover']}"'
        onmouseout='this.style.background="{COLORS['success']}"'>
            🔄 Hitung Lagi
        </button>
        
        <button onclick='window.scrollTo(0,0)' style='
            background: {COLORS['theory']}; 
            color: white; 
            border: none; 
            padding: 15px; 
            border-radius: 10px; 
            font-family: {FONT}; 
            font-size: 16px; 
            cursor: pointer;
            transition: all 0.3s;
        ' onmouseover='this.style.background="{COLORS['button_hover']}"'
        onmouseout='this.style.background="{COLORS['theory']}"'>
            ⬆️ Ke Atas
        </button>
    </div>
    """)

if __name__ == '__main__':
    start_server(main, port=8080, debug=True)
