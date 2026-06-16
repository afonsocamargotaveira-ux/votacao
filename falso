"""
App Streamlit - Como um Deputado Votou
========================================

Consulta a API de Dados Abertos da Câmara dos Deputados
(https://dadosabertos.camara.leg.br/swagger/api.html) para
descobrir como um determinado deputado votou em votações
nominais (abertas) do Plenário ou de Comissões.

Funcionalidades:
- Busca de deputado por nome (com autocomplete via lista retornada da API)
- Busca de votações por:
    a) Proposição (ex: PL 1234/2023)
    b) Período de datas
- Exibe, para cada votação encontrada:
    - Proposição (sigla, número, ano, ementa)
    - Data e hora da votação
    - Órgão (Plenário, comissão, etc.)
    - Descrição da votação
    - Resultado (Aprovada/Rejeitada)
    - O voto do deputado selecionado (Sim, Não, Abstenção, Obstrução, Ausente etc.)
    - Orientação do partido/bloco do deputado (quando disponível)

Como executar:
    pip install streamlit requests pandas
    streamlit run app_votos_deputados.py
"""

import streamlit as st
import requests
import pandas as pd
import io

BASE_URL = "https://dadosabertos.camara.leg.br/api/v2"
HEADERS = {"accept": "application/json"}

st.set_page_config(
    page_title="Como o Deputado Votou?",
    page_icon="🏛️",
    layout="wide",
)

# ----------------------------------------------------------------------
# Tema visual (azul marinho / laranja / branco)
# ----------------------------------------------------------------------
st.markdown(
    """
    <style>
        :root {
            --navy: #0B1F3A;
            --navy-light: #14304F;
            --orange: #FF7A1A;
            --white: #FFFFFF;
        }

        /* Fundo geral */
        .stApp {
            background-color: var(--navy);
            color: var(--white);
        }

        /* Sidebar (se usada) */
        section[data-testid="stSidebar"] {
            background-color: var(--navy-light);
        }

        /* Títulos */
        h1, h2, h3, h4, h5, h6 {
            color: var(--white) !important;
        }
        h1 {
            border-bottom: 3px solid var(--orange);
            padding-bottom: 0.4rem;
        }

        /* Texto e markdown padrão */
        .stMarkdown, .stCaption, p, label, span, div {
            color: var(--white);
        }

        /* Divisores */
        hr {
            border-color: var(--orange) !important;
            opacity: 0.6;
        }

        /* Inputs de texto */
        .stTextInput input, .stSelectbox div[data-baseweb="select"] > div {
            background-color: var(--navy-light) !important;
            color: var(--white) !important;
            border: 1px solid var(--orange) !important;
            border-radius: 6px;
        }

        /* Botões */
        .stButton > button, .stDownloadButton > button {
            background-color: var(--orange) !important;
            color: var(--navy) !important;
            border: none !important;
            border-radius: 6px;
            font-weight: 700;
            transition: all 0.15s ease-in-out;
        }
        .stButton > button:hover, .stDownloadButton > button:hover {
            background-color: var(--white) !important;
            color: var(--navy) !important;
            box-shadow: 0 0 0 2px var(--orange) inset;
        }

        /* Slider */
        .stSlider [data-baseweb="slider"] > div > div {
            background-color: var(--orange) !important;
        }

        /* Containers / cards */
        div[data-testid="stVerticalBlockBorderWrapper"] {
            background-color: var(--navy-light);
            border: 1px solid var(--orange) !important;
            border-radius: 10px;
            padding: 0.5rem;
        }

        /* Métricas */
        div[data-testid="stMetric"] {
            background-color: var(--navy);
            border: 1px solid var(--orange);
            border-radius: 8px;
            padding: 0.6rem;
            text-align: center;
        }
        div[data-testid="stMetricValue"] {
            color: var(--orange) !important;
            font-weight: 800;
        }
        div[data-testid="stMetricLabel"] {
            color: var(--white) !important;
        }

        /* Expander */
        details {
            background-color: var(--navy);
            border: 1px solid var(--orange);
            border-radius: 8px;
        }
        summary {
            color: var(--orange) !important;
            font-weight: 600;
        }

        /* Links */
        a, a:visited {
            color: var(--orange) !important;
            font-weight: 600;
        }

        /* Alertas (info/warning/error/success) - mantém legibilidade */
        div[data-testid="stAlert"] {
            border-radius: 8px;
            border: 1px solid var(--orange);
        }

        /* Imagem do deputado com moldura */
        div[data-testid="stImage"] img {
            border: 3px solid var(--orange);
            border-radius: 8px;
        }
    </style>
    """,
    unsafe_allow_html=True,
)

# ----------------------------------------------------------------------
# Funções auxiliares de acesso à API (com cache para evitar requisições repetidas)
# ----------------------------------------------------------------------

@st.cache_data(show_spinner=False, ttl=3600)
def buscar_deputados(nome: str):
    """Busca deputados (atuais ou já atuantes na legislatura) pelo nome."""
    if not nome or len(nome) < 3:
        return []
    params = {"nome": nome, "itens": 50, "ordem": "ASC", "ordenarPor": "nome"}
    try:
        r = requests.get(f"{BASE_URL}/deputados", params=params, headers=HEADERS, timeout=20)
        r.raise_for_status()
        return r.json().get("dados", [])
    except requests.RequestException as e:
        st.error(f"Erro ao buscar deputados: {e}")
        return []


@st.cache_data(show_spinner=False, ttl=3600)
def buscar_proposicoes(sigla: str, numero, ano):
    """Busca proposições por sigla/número/ano."""
    params = {"itens": 20}
    if sigla:
        params["siglaTipo"] = sigla
    if numero:
        params["numero"] = numero
    if ano:
        params["ano"] = ano
    try:
        r = requests.get(f"{BASE_URL}/proposicoes", params=params, headers=HEADERS, timeout=20)
        r.raise_for_status()
        return r.json().get("dados", [])
    except requests.RequestException as e:
        st.error(f"Erro ao buscar proposições: {e}")
        return []


@st.cache_data(show_spinner=False, ttl=3600)
def buscar_proposicoes_relacionadas(id_proposicao: int):
    """Lista proposições relacionadas (substitutivos, redações finais, apensados etc.)."""
    try:
        r = requests.get(
            f"{BASE_URL}/proposicoes/{id_proposicao}/relacionadas",
            headers=HEADERS,
            timeout=20,
        )
        r.raise_for_status()
        return r.json().get("dados", [])
    except requests.RequestException:
        return []


@st.cache_data(show_spinner=False, ttl=3600)
def buscar_votacoes_por_proposicao(id_proposicao: int):
    """Lista as votações relacionadas a uma proposição."""
    try:
        r = requests.get(
            f"{BASE_URL}/proposicoes/{id_proposicao}/votacoes",
            headers=HEADERS,
            timeout=20,
        )
        r.raise_for_status()
        return r.json().get("dados", [])
    except requests.RequestException as e:
        try:
            detalhe = r.json()
        except Exception:
            detalhe = getattr(e, "response", None) and e.response.text
        st.error(f"Erro ao buscar votações da proposição: {e}\n\nDetalhe da API: {detalhe}")
        return []


@st.cache_data(show_spinner=False, ttl=3600)
def detalhar_votacao(id_votacao: str):
    """Detalhes de uma votação específica (inclui proposições afetadas)."""
    try:
        r = requests.get(f"{BASE_URL}/votacoes/{id_votacao}", headers=HEADERS, timeout=20)
        r.raise_for_status()
        return r.json().get("dados", {})
    except requests.RequestException:
        return {}


@st.cache_data(show_spinner=False, ttl=3600)
def buscar_votos(id_votacao: str):
    """Retorna a lista de votos individuais de uma votação nominal."""
    todos = []
    pagina = 1
    try:
        while True:
            r = requests.get(
                f"{BASE_URL}/votacoes/{id_votacao}/votos",
                headers=HEADERS,
                timeout=30,
            )
            r.raise_for_status()
            dados = r.json().get("dados", [])
            todos.extend(dados)
            break
        return todos
    except requests.RequestException as e:
        st.warning(f"Erro ao buscar votos da votação {id_votacao}: {e}")
        return todos


@st.cache_data(show_spinner=False, ttl=3600)
def buscar_orientacoes(id_votacao: str):
    """Retorna as orientações de bancada para uma votação."""
    try:
        r = requests.get(
            f"{BASE_URL}/votacoes/{id_votacao}/orientacoes",
            headers=HEADERS,
            timeout=20,
        )
        r.raise_for_status()
        return r.json().get("dados", [])
    except requests.RequestException:
        return []


def formatar_proposicao(prop) -> str:
    if not prop:
        return "—"
    if isinstance(prop, list):
        if not prop:
            return "—"
        prop = prop[0]
    if not isinstance(prop, dict):
        return str(prop)

    # Formato vindo de /proposicoes (siglaTipo/numero/ano/ementa)
    if "siglaTipo" in prop:
        sigla = prop.get("siglaTipo", "")
        numero = prop.get("numero", "")
        ano = prop.get("ano", "")
        ementa = prop.get("ementa", "")
        base = f"{sigla} {numero}/{ano}".strip()
        if ementa:
            return f"{base} — {ementa}"
        return base

    # Formato vindo de /votacoes -> ultimaApresentacaoProposicao
    if "descricao" in prop:
        return prop.get("descricao", "—")

    return "—"


def voto_emoji(voto: str) -> str:
    mapa = {
        "Sim": "✅ Sim",
        "Não": "❌ Não",
        "Abstenção": "⚪ Abstenção",
        "Obstrução": "🚫 Obstrução",
        "Artigo 17": "📜 Art. 17",
    }
    return mapa.get(voto, voto if voto else "—")


# ----------------------------------------------------------------------
# Interface
# ----------------------------------------------------------------------

st.title("🏛️ Como o Deputado Votou?")
st.markdown(
    "Consulte como um(a) **deputado(a) federal** votou — ou veja **todos os votos** — "
    "na votação nominal (aberta) principal de uma proposição, usando o "
    "[Portal de Dados Abertos da Câmara](https://dadosabertos.camara.leg.br/)."
)

st.divider()

# --- 1) Modo de consulta -----------------------------------------------------
st.subheader("1️⃣ O que você quer consultar?")

modo_consulta = st.radio(
    "Modo de consulta:",
    ["Voto de um(a) deputado(a) específico(a)", "Todos os votos da votação principal"],
    horizontal=True,
)

deputado_selecionado = None

if modo_consulta == "Voto de um(a) deputado(a) específico(a)":
    st.markdown("**Escolha o(a) deputado(a)**")
    col1, col2 = st.columns([2, 1])
    with col1:
        nome_busca = st.text_input(
            "Digite o nome (ou parte do nome) do deputado",
            placeholder="Ex: Tabata Amaral, Nikolas Ferreira, Lula da Silva...",
        )

    if nome_busca and len(nome_busca) >= 3:
        resultados = buscar_deputados(nome_busca)
        if resultados:
            opcoes = {
                f'{d["nome"]} — {d.get("siglaPartido","")}/{d.get("siglaUf","")} (id {d["id"]})': d
                for d in resultados
            }
            escolha = st.selectbox("Selecione o deputado encontrado:", list(opcoes.keys()))
            deputado_selecionado = opcoes[escolha]

            with col2:
                if deputado_selecionado.get("urlFoto"):
                    st.image(deputado_selecionado["urlFoto"], width=120)
        else:
            st.info("Nenhum deputado encontrado com esse nome.")
    else:
        st.caption("Digite ao menos 3 letras para buscar.")

pode_prosseguir = (
    modo_consulta == "Todos os votos da votação principal" or deputado_selecionado is not None
)

st.divider()

# --- 2) Busca da votação por proposição --------------------------------------
votacoes_encontradas = []
prop_sel = None

if pode_prosseguir:
    st.subheader("2️⃣ Informe a proposição")

    st.markdown("**Informe os dados da proposição** (ex: PL 1234/2023)")
    c1, c2, c3 = st.columns(3)
    with c1:
        sigla = st.text_input("Sigla do tipo", value="PL", help="Ex: PL, PEC, MPV, PLP, PDL...")
    with c2:
        numero = st.text_input("Número", placeholder="Ex: 1234")
    with c3:
        ano = st.text_input("Ano", placeholder="Ex: 2023")

    if st.button("🔍 Buscar proposição"):
        if not numero or not ano:
            st.warning("Informe ao menos o número e o ano da proposição.")
        else:
            props = buscar_proposicoes(sigla.upper().strip(), numero.strip(), ano.strip())
            if not props:
                st.warning("Nenhuma proposição encontrada com esses dados.")
            else:
                st.session_state["props_encontradas"] = props

    if "props_encontradas" in st.session_state and st.session_state["props_encontradas"]:
        props = st.session_state["props_encontradas"]
        opcoes_prop = {formatar_proposicao(p): p for p in props}
        escolha_prop = st.selectbox("Selecione a proposição:", list(opcoes_prop.keys()))
        prop_sel = opcoes_prop[escolha_prop]

        todas_votacoes = buscar_votacoes_por_proposicao(prop_sel["id"])

        if not todas_votacoes:
            relacionadas = buscar_proposicoes_relacionadas(prop_sel["id"])
            for rel in relacionadas:
                rel_id = rel.get("id")
                if not rel_id:
                    continue
                vot_rel = buscar_votacoes_por_proposicao(rel_id)
                if vot_rel:
                    todas_votacoes.extend(vot_rel)

        sigla_prop = prop_sel.get("siglaTipo", "")
        nome_tipo = {
            "PEC": "Proposta de Emenda à Constituição",
            "PL": "Projeto de Lei",
            "PLP": "Projeto de Lei Complementar",
            "PDL": "Projeto de Decreto Legislativo",
            "MPV": "Medida Provisória",
        }.get(sigla_prop, "")

        def eh_votacao_principal(v):
            desc = (v.get("descricao") or "").lower()
            if "redação final" in desc:
                return False
            if "destaque" in desc:
                return False
            if "emenda de plenário" in desc:
                return False
            if "requerimento" in desc:
                return False
            if ("aprovad" in desc or "rejeitad" in desc or "mantid" in desc) and (
                not nome_tipo or nome_tipo.lower() in desc or str(prop_sel.get("numero", "")) in desc
            ):
                return True
            return False

        votacoes_encontradas = [v for v in todas_votacoes if eh_votacao_principal(v)]

        if not votacoes_encontradas and todas_votacoes:
            st.warning(
                "Não foi possível identificar automaticamente a votação principal "
                "(aprovação do mérito) desta proposição. Selecione manualmente a votação "
                "correta na lista abaixo:"
            )
            opcoes_vot = {
                f'{v.get("dataHoraRegistro","")} — {v.get("siglaOrgao","")} — '
                f'{(v.get("descricao") or "")[:120]}': v
                for v in todas_votacoes
            }
            escolha_vot = st.selectbox("Votações nominais encontradas para esta proposição:", list(opcoes_vot.keys()))
            votacoes_encontradas = [opcoes_vot[escolha_vot]]
        elif not votacoes_encontradas:
            st.info("Não há votações nominais registradas para esta proposição.")

    # --- Alternativa: buscar diretamente pelo ID da votação ------------------
    with st.expander("🔧 Alternativa: buscar diretamente pelo ID da votação"):
        st.caption(
            "Use esta opção se souber o ID da votação na API "
            "(ex: 2256337-43, no formato idEvento-idOrgao, ou apenas o ideVotacao antigo)."
        )
        id_votacao_manual = st.text_input("ID da votação", placeholder="Ex: 2256337-43")
        if st.button("Buscar por ID da votação") and id_votacao_manual.strip():
            votacoes_encontradas = [{"id": id_votacao_manual.strip()}]
            prop_sel = None

    # --- 3) Resultados -------------------------------------------------------
    if votacoes_encontradas:
        st.divider()
        st.subheader("3️⃣ Resultados")

        if len(votacoes_encontradas) > 1:
            max_exibir = st.slider(
                "Quantidade máxima de votações a analisar (mais votações = mais lento)",
                min_value=1,
                max_value=min(50, len(votacoes_encontradas)),
                value=min(10, len(votacoes_encontradas)),
            )
        else:
            max_exibir = 1

        votacoes_para_analisar = votacoes_encontradas[:max_exibir]

        progresso = st.progress(0.0)

        for i, vot in enumerate(votacoes_para_analisar):
            id_vot = vot.get("id")
            detalhes = detalhar_votacao(id_vot)

            # Proposição relacionada (quando existir)
            prop_obj = None
            if detalhes.get("ultimaApresentacaoProposicao"):
                prop_obj = detalhes["ultimaApresentacaoProposicao"]
            elif detalhes.get("proposicaoObjeto"):
                prop_obj = detalhes["proposicaoObjeto"]
            elif vot.get("proposicaoObjeto"):
                prop_obj = vot.get("proposicaoObjeto")

            descricao = detalhes.get("descricao") or vot.get("descricao", "")
            data_hora = detalhes.get("dataHoraRegistro") or vot.get("dataHoraRegistro", "")
            orgao = (detalhes.get("siglaOrgao") or vot.get("siglaOrgao", ""))
            aprovada = detalhes.get("aprovacao", vot.get("aprovacao"))
            if aprovada == 1:
                resultado_txt = "✅ Aprovada"
            elif aprovada == 0:
                resultado_txt = "❌ Rejeitada"
            else:
                resultado_txt = "—"

            id_proposicao_link = None
            if isinstance(prop_obj, dict) and "idProposicao" in prop_obj:
                id_proposicao_link = prop_obj.get("idProposicao")
            elif prop_sel:
                id_proposicao_link = prop_sel.get("id")

            proposicao_txt = formatar_proposicao(prop_obj) if prop_obj else "—"

            votos = buscar_votos(id_vot)

            with st.container(border=True):
                st.markdown(f"**Proposição:** {proposicao_txt}")
                st.markdown(f"**Votação:** {descricao}")
                st.caption(f"Órgão: {orgao}  |  Data/Hora: {data_hora}")
                st.caption(f"Resultado da votação: {resultado_txt}  |  Total de votos registrados: {len(votos)}")

                # ----------------------------------------------------
                # MODO: voto de um deputado específico
                # ----------------------------------------------------
                if modo_consulta == "Voto de um(a) deputado(a) específico(a)":
                    voto_dep = None
                    for v in votos:
                        dep_info = v.get("deputado_", {})
                        if dep_info.get("id") == deputado_selecionado["id"]:
                            voto_dep = v.get("tipoVoto")
                            break

                    orientacao_partido = "—"
                    if voto_dep is not None:
                        orientacoes = buscar_orientacoes(id_vot)
                        sigla_partido_dep = deputado_selecionado.get("siglaPartido", "")
                        for o in orientacoes:
                            if o.get("siglaPartidoBloco", "") == sigla_partido_dep or sigla_partido_dep in o.get(
                                "siglaPartidoBloco", ""
                            ):
                                orientacao_partido = o.get("orientacaoVoto", "—")
                                break

                    colA, colB = st.columns([3, 1])
                    with colB:
                        st.metric(
                            label=f"Voto de {deputado_selecionado['nome']}",
                            value=voto_emoji(voto_dep) if voto_dep else "🔘 Não votou / Ausente",
                        )
                        st.caption(
                            f"Orientação do partido ({deputado_selecionado.get('siglaPartido','')}): "
                            f"{orientacao_partido}"
                        )

                # ----------------------------------------------------
                # MODO: todos os votos
                # ----------------------------------------------------
                else:
                    if not votos:
                        st.warning(
                            "Nenhum voto individual foi retornado pela API para esta votação. "
                            "Isso geralmente significa que ela foi **simbólica** (decidida por "
                            "acordo de lideranças, sem registro de voto de cada parlamentar), "
                            "ou que o ID da votação não corresponde a uma votação nominal válida."
                        )
                        st.caption(
                            f"Você pode verificar manualmente os dados brutos desta votação em: "
                            f"https://dadosabertos.camara.leg.br/api/v2/votacoes/{id_vot}"
                        )
                    else:
                        # Resumo por tipo de voto
                        resumo = {}
                        for v in votos:
                            tv = v.get("tipoVoto", "—")
                            resumo[tv] = resumo.get(tv, 0) + 1

                        cols_resumo = st.columns(len(resumo) if resumo else 1)
                        for (tv, qtd), c in zip(resumo.items(), cols_resumo):
                            with c:
                                st.metric(label=voto_emoji(tv), value=qtd)

                        # Tabela completa com todos os deputados
                        linhas_votos = []
                        for v in votos:
                            dep_info = v.get("deputado_", {}) or {}
                            linhas_votos.append(
                                {
                                    "Deputado": dep_info.get("nome", "—"),
                                    "Partido": dep_info.get("siglaPartido", "—"),
                                    "UF": dep_info.get("siglaUf", "—"),
                                    "Voto": voto_emoji(v.get("tipoVoto")),
                                }
                            )

                        df_votos = pd.DataFrame(linhas_votos).sort_values(
                            by=["Voto", "Deputado"]
                        ).reset_index(drop=True)

                        # Filtro de busca dentro da tabela de votos
                        filtro_nome = st.text_input(
                            "Filtrar por nome do deputado nesta votação:",
                            key=f"filtro_{id_vot}",
                            placeholder="Ex: Tabata, Nikolas...",
                        )
                        if filtro_nome:
                            df_exibir = df_votos[
                                df_votos["Deputado"].str.contains(filtro_nome, case=False, na=False)
                            ]
                        else:
                            df_exibir = df_votos

                        st.dataframe(df_exibir, use_container_width=True, hide_index=True)

                        buffer_xlsx = io.BytesIO()
                        with pd.ExcelWriter(buffer_xlsx, engine="openpyxl") as writer:
                            df_votos.to_excel(writer, index=False, sheet_name="Votos")
                        st.download_button(
                            "📥 Baixar todos os votos desta votação (Excel)",
                            data=buffer_xlsx.getvalue(),
                            file_name=f"votos_votacao_{id_vot}.xlsx",
                            mime="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                            key=f"download_{id_vot}",
                        )

                with st.expander("Ver detalhes técnicos / link da votação"):
                    st.write(f"ID da votação: `{id_vot}`")
                    if id_proposicao_link:
                        st.markdown(
                            f"[Abrir proposição no portal da Câmara]"
                            f"(https://www.camara.leg.br/proposicoesWeb/fichadetramitacao?idProposicao={id_proposicao_link})"
                        )
                    else:
                        st.caption("Link da proposição não disponível para esta votação.")

            progresso.progress((i + 1) / len(votacoes_para_analisar))

        progresso.empty()

else:
    st.info("⬆️ Selecione o modo de consulta (e, se aplicável, o deputado) para continuar.")

st.divider()
st.caption(
    "Fonte dos dados: API de Dados Abertos da Câmara dos Deputados "
    "(https://dadosabertos.camara.leg.br/). "
    "Apenas votações **nominais (abertas)** registram o voto individual de cada parlamentar; "
    "votações simbólicas (de liderança) não possuem voto individual disponível."
)
