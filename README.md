36.DRE Assistente HTML = VAR Pergunta =
    SELECTEDVALUE('dPerguntasDRE'[Ordem], 0)

VAR TextoPergunta =
    SELECTEDVALUE('dPerguntasDRE'[Pergunta], "")

VAR MesUnico =
    HASONEVALUE('Calendário'[AnoMês])

VAR DataReferencia =
    MAX('Calendário'[Date])

VAR MesAtual =
    FORMAT(DataReferencia, "MMM/yy", "pt-BR")

VAR MesAnterior =
    FORMAT(
        EOMONTH(DataReferencia, -1),
        "MMM/yy",
        "pt-BR"
    )

VAR Cenario =
    SELECTEDVALUE(
        'dCenarioDRE'[Cenario],
        "Realizado + Projetado"
    )

VAR LinhaAlvo =
    SWITCH(
        Pergunta,
        2, "CUSTOS DOS SERVIÇOS ( - )",
        4, "DESPESAS OPERACIONAIS ( - )",
        BLANK()
    )

VAR LinhasResumo =
    FILTER(
        ALL('dDRE'),
        SWITCH(
            Pergunta,
            1,
                'dDRE'[Ordem] IN { 1, 5, 7, 8, 12, 18 },
            3,
                'dDRE'[Ordem] IN { 1, 4, 5, 7, 8 },
            FALSE()
        )
    )

VAR ResumoValores =
    ADDCOLUMNS(
        LinhasResumo,
        "@Atual",
            VAR Linha =
                'dDRE'[Linha]
            RETURN
                CALCULATE(
                    [18.DRE Gerencial de Caixa],
                    REMOVEFILTERS('dDRE'),
                    'dDRE'[Linha] = Linha
                ),
        "@Anterior",
            VAR Linha =
                'dDRE'[Linha]
            RETURN
                CALCULATE(
                    [30.DRE Mês Anterior],
                    REMOVEFILTERS('dDRE'),
                    'dDRE'[Linha] = Linha
                )
    )

VAR Resumo =
    SELECTCOLUMNS(
        ResumoValores,
        "@Nome", 'dDRE'[Linha],
        "@Ordem", 'dDRE'[Ordem],
        "@Valor", [@Atual],
        "@Base", [@Anterior]
    )

VAR Planos =
    CALCULATETABLE(
        VALUES('fBase'[PLANO CONTAS]),
        REMOVEFILTERS('Calendário'),
        REMOVEFILTERS('dDRE')
    )

VAR PlanosValores =
    ADDCOLUMNS(
        Planos,
        "@Atual",
            CALCULATE(
                [18.DRE Gerencial de Caixa],
                REMOVEFILTERS('dDRE'),
                'dDRE'[Linha] = LinhaAlvo
            ),
        "@Anterior",
            CALCULATE(
                [30.DRE Mês Anterior],
                REMOVEFILTERS('dDRE'),
                'dDRE'[Linha] = LinhaAlvo
            )
    )

VAR PlanosVariacao =
    ADDCOLUMNS(
        PlanosValores,
        "@Aumento",
            COALESCE([@Anterior], 0)
                - COALESCE([@Atual], 0)
    )

VAR TopAumentos =
    TOPN(
        3,
        FILTER(
            PlanosVariacao,
            [@Aumento] > 0.005
        ),
        [@Aumento], DESC,
        'fBase'[PLANO CONTAS], ASC
    )

VAR Detalhes =
    SELECTCOLUMNS(
        TopAumentos,
        "@Nome",
            COALESCE(
                'fBase'[PLANO CONTAS],
                IF(Pergunta = 2, "Custo sem identificação", "Despesa sem identificação")
            ),
        "@Ordem", -[@Aumento],
        "@Valor", [@Atual],
        "@Base", [@Anterior]
    )

VAR Dados =
    UNION(
        FILTER(
            Resumo,
            Pergunta IN { 1, 3 }
        ),
        FILTER(
            Detalhes,
            Pergunta IN { 2, 4 }
        )
    )

VAR LinhasHTML =
    CONCATENATEX(
        Dados,
        VAR NomeSeguro = SUBSTITUTE(SUBSTITUTE(SUBSTITUTE([@Nome], "&", "&amp;"), "<", "&lt;"), ">", "&gt;")
        VAR Diferenca = COALESCE([@Valor], 0) - COALESCE([@Base], 0)
        VAR SemValores = ISBLANK([@Valor]) && ISBLANK([@Base])
        VAR Percentual = DIVIDE(Diferenca, ABS([@Base]))
        VAR Cor = SWITCH(TRUE(), Diferenca > 0.005, "#007A55", Diferenca < -0.005, "#C62828", "#64748B")
        VAR EhGasto =
            Pergunta IN { 2, 4 }
                || [@Nome] IN {
                    "CUSTOS DOS SERVIÇOS ( - )",
                    "DESPESAS OPERACIONAIS ( - )",
                    "DESP. FINANCEIRA ( - )",
                    "IMPOSTOS SOB RECEITAS ( - )"
                }
        VAR Movimento = IF(EhGasto, -Diferenca, Diferenca)
        VAR Seta = SWITCH(TRUE(), Movimento > 0.005, "↑", Movimento < -0.005, "↓", "→")
        VAR DicaSeta =
            SWITCH(TRUE(),
                ABS(Movimento) <= 0.005, "Sem alteração",
                EhGasto && Movimento > 0, "O gasto aumentou",
                EhGasto && Movimento < 0, "O gasto diminuiu",
                Movimento > 0, "O indicador aumentou",
                "O indicador diminuiu"
            )
        VAR Estilo = "padding:9px 6px;text-align:right;white-space:nowrap;border-bottom:1px solid #E5E7EB;"
        RETURN
            "<tr><td style='padding:9px 6px;border-bottom:1px solid #E5E7EB;'>" & NomeSeguro
            & "</td><td style='" & Estilo & "background:#F8FAFC;'>"
            & IF(ISBLANK([@Base]), "—", FORMAT([@Base], "#,##0.00", "pt-BR"))
            & "</td><td style='" & Estilo & "font-weight:600;'>"
            & IF(ISBLANK([@Valor]), "—", FORMAT([@Valor], "#,##0.00", "pt-BR"))
            & "</td><td style='" & Estilo & "color:" & Cor & ";'>"
            & IF(SemValores, "—", IF(Diferenca > 0, "+", "") & FORMAT(Diferenca, "#,##0.00", "pt-BR"))
            & "</td><td style='" & Estilo & "color:" & Cor & ";'>"
            & IF(
                SemValores,
                "—",
                "<span title='" & DicaSeta & "' style='font-weight:700;font-size:15px;'>" & Seta & "</span>&nbsp;"
                & IF(ISBLANK(Percentual), "—", FORMAT(ABS(Percentual), "0.00%", "pt-BR"))
            )
            & "</td></tr>",
        "", [@Ordem], ASC
    )
VAR LinhaResultado =
    IF(
        Pergunta = 1,
        "SALDO DOS MOVIMENTOS APRESENTADOS",
        "RESULTADO APÓS DESPESAS OPERACIONAIS"
    )

VAR ResultadoAtual =
    CALCULATE(
        [18.DRE Gerencial de Caixa],
        REMOVEFILTERS('dDRE'),
        'dDRE'[Linha] = LinhaResultado
    )

VAR ResultadoAnterior =
    CALCULATE(
        [30.DRE Mês Anterior],
        REMOVEFILTERS('dDRE'),
        'dDRE'[Linha] = LinhaResultado
    )

VAR DiferencaResultado =
    COALESCE(ResultadoAtual, 0)
        - COALESCE(ResultadoAnterior, 0)

VAR PercentualResultado = DIVIDE(DiferencaResultado, ABS(ResultadoAnterior))

VAR TextoResultado =
    IF(Pergunta = 1, "O saldo dos movimentos apresentados", "O resultado após despesas operacionais")
    & " registrou <b>"
    & IF(ISBLANK(ResultadoAnterior), "sem valor disponível", "R$ " & FORMAT(ResultadoAnterior, "#,##0.00", "pt-BR"))
    & " em " & MesAnterior & "</b> e <b>"
    & IF(ISBLANK(ResultadoAtual), "sem valor disponível", "R$ " & FORMAT(ResultadoAtual, "#,##0.00", "pt-BR"))
    & " em " & MesAtual & "</b>. "
    & IF(
        ISBLANK(ResultadoAnterior) || ISBLANK(ResultadoAtual),
        "Um dos períodos não possui valor disponível; confira a base antes de interpretar a variação.",
        SWITCH(
            TRUE(),
            ABS(DiferencaResultado) < 0.005, "Não houve variação entre os períodos.",
            IF(DiferencaResultado > 0, "Houve melhora de <b>R$ ", "Houve queda de <b>R$ ")
                & FORMAT(ABS(DiferencaResultado), "#,##0.00", "pt-BR") & "</b>"
                & IF(ISBLANK(PercentualResultado), ". A base anterior é zero; percentual não calculado.",
                    " (<b>" & FORMAT(ABS(PercentualResultado), "0.00%", "pt-BR") & "</b> sobre o valor absoluto do mês anterior).")
        )
    )
VAR TotalGrupoAtual =
    CALCULATE(
        [18.DRE Gerencial de Caixa],
        REMOVEFILTERS('dDRE'),
        'dDRE'[Linha] = LinhaAlvo
    )

VAR TotalGrupoAnterior =
    CALCULATE(
        [30.DRE Mês Anterior],
        REMOVEFILTERS('dDRE'),
        'dDRE'[Linha] = LinhaAlvo
    )

VAR SemDadosGrupo =
    ISBLANK(TotalGrupoAtual)
        && ISBLANK(TotalGrupoAnterior)

VAR VariacaoGastoGrupo =
    COALESCE(TotalGrupoAnterior, 0)
        - COALESCE(TotalGrupoAtual, 0)

VAR PercentualGrupo =
    DIVIDE(
        ABS(VariacaoGastoGrupo),
        ABS(TotalGrupoAnterior)
    )

VAR NomeGrupo =
    IF(
        Pergunta = 2,
        "Os custos dos serviços",
        "As despesas operacionais"
    )

VAR TextoPercentual =
    IF(
        NOT ISBLANK(PercentualGrupo),
        " ("
            & FORMAT(
                PercentualGrupo,
                "0.00%",
                "pt-BR"
            )
            & ")",
        ""
    )

VAR TextoTotalGrupo =
    IF(
        SemDadosGrupo,
        "Não há valores para este grupo nos períodos selecionados.",
        NomeGrupo & " registraram <b>"
        & IF(ISBLANK(TotalGrupoAnterior), "sem valor disponível", "R$ " & FORMAT(TotalGrupoAnterior, "#,##0.00", "pt-BR"))
        & " em " & MesAnterior & "</b> e <b>"
        & IF(ISBLANK(TotalGrupoAtual), "sem valor disponível", "R$ " & FORMAT(TotalGrupoAtual, "#,##0.00", "pt-BR"))
        & " em " & MesAtual & "</b>. "
        & SWITCH(
            TRUE(),
            ABS(VariacaoGastoGrupo) < 0.005, "O gasto total ficou estável.",
            IF(VariacaoGastoGrupo > 0, "O gasto total <b>aumentou R$ ", "O gasto total <b>diminuiu R$ ")
                & FORMAT(ABS(VariacaoGastoGrupo), "#,##0.00", "pt-BR") & TextoPercentual & "</b>."
        )
    )
VAR TotalReducoes =
    -SUMX(
        FILTER(
            PlanosVariacao,
            [@Aumento] < -0.005
        ),
        [@Aumento]
    )

VAR TextoRanking =
    SWITCH(
        TRUE(),
        COUNTROWS(TopAumentos) = 0,
            IF(Pergunta = 2,
                " Nenhum custo apresentou aumento.",
                " Nenhuma despesa apresentou aumento."
            ),
        VariacaoGastoGrupo < -0.005,
            IF(Pergunta = 2,
                " Apesar da redução total, alguns custos aumentaram. A tabela destaca os custos com os maiores aumentos.",
                " Apesar da redução total, algumas despesas aumentaram. A tabela destaca as despesas com os maiores aumentos."
            ),
        IF(Pergunta = 2,
            " A tabela destaca os custos com os maiores aumentos.",
            " A tabela destaca as despesas com os maiores aumentos."
        )
    )

VAR TextoCompensacoes =
    IF(
        TotalReducoes > 0.005,
        IF(Pergunta = 2,
            " Os custos que diminuíram somaram uma redução de <b>R$ ",
            " As despesas que diminuíram somaram uma redução de <b>R$ "
        )
        & FORMAT(TotalReducoes, "#,##0.00", "pt-BR")
        & "</b>.",
        ""
    )
VAR TextoIntroducao =
    SWITCH(
        TRUE(),

        Pergunta IN { 1, 3 },
            IF(
                ISBLANK(ResultadoAtual)
                    && ISBLANK(ResultadoAnterior),
                "Não há resultado disponível para os filtros selecionados.",
                TextoResultado
            ),

        SemDadosGrupo,
            TextoTotalGrupo,

        TextoTotalGrupo
            & TextoRanking
            & TextoCompensacoes
            & IF(
                COUNTROWS(TopAumentos) > 0,
                "<br><br>
                 <span style='font-size:11px;color:#64748B;'>
                    Na tabela, variação negativa significa aumento do gasto.
                 </span>",
                ""
            )
    )

VAR Conteudo =
    SWITCH(
        TRUE(),

             Pergunta = 0,
            "<div style='text-align:center;padding:24px 4px 0;'>

                <div style='
                    width:56px;
                    height:56px;
                    margin:0 auto 24px;
                    background:#E3EBF4;
                    border-radius:50%;
                    padding:8px;
                    box-sizing:border-box;
                '>
                    <div style='
                        width:40px;
                        height:40px;
                        background:#2D4868;
                        border:4px solid #FFFFFF;
                        border-radius:50%;
                        box-sizing:border-box;
                        color:#FFFFFF;
                        font-size:27px;
                        line-height:32px;
                        text-align:center;
                    '>✦</div>
                </div>

                <div style='
                    font-size:22px;
                    font-weight:600;
                    line-height:1.3;
                    color:#111827;
                    margin-bottom:12px;
                '>
                    Como posso te ajudar hoje?
                </div>

                <div style='
                    font-size:12px;
                    line-height:1.7;
                    color:#365779;
                    margin:0 auto 22px;
                    max-width:360px;
                '>
                    Consulte receitas, custos, despesas e resultados
                    da Medic Services.
                </div>

                <div style='
                    border:1px solid #DCE1EA;
                    border-radius:10px;
                    padding:16px;
                    text-align:left;
                    background:#FFFFFF;
                '>
                    <div style='
                        font-size:12px;
                        color:#365779;
                        margin-bottom:12px;
                    '>
                        O que você gostaria de analisar?
                    </div>

                    <div style='
                        font-size:11px;
                        color:#64748B;
                    '>
                        Escolha uma das perguntas abaixo.
                        A resposta considera os filtros do relatório.
                    </div>
                </div>

                

            </div>",

        NOT MesUnico,
            "<p>
                Selecione <b>um único mês e ano</b>
                para comparar com o mês anterior.
             </p>",

        "<div style='text-align:right;margin-bottom:16px;'>
            <span style='
                display:inline-block;
                padding:10px 12px;
                background:#F1F5F9;
                border:1px solid #DCE3ED;
                border-radius:10px;
            '>"
                & TextoPergunta &
            "</span>
         </div>

         <div style='
            font-size:11px;
            color:#64748B;
            margin-bottom:12px;
         '>"
            & MesAtual
            & " versus "
            & MesAnterior
            & " | "
            & Cenario &
         "</div>

         <p style='margin:0 0 16px;'>"
            & TextoIntroducao &
         "</p>"

            & IF(
                COUNTROWS(Dados) > 0,
                "<div style='
                    border:1px solid #DCE3ED;
                    border-radius:10px;
                    padding:10px;overflow-x:auto;
                '>
                    <table style='
                        width:100%;
                        border-collapse:collapse;min-width:560px;
                        font-size:11px;
                    '>
                        <thead>
                            <tr>
                                <th style='text-align:left;padding:8px 4px;'>
                                    Indicador
                                </th>
                                <th style='text-align:right;padding:8px 4px;background:#F8FAFC;'>"
                                    & MesAnterior &
                                "</th>
                                <th style='text-align:right;padding:8px 4px;'>"
                                    & MesAtual &
                                "</th>
                                <th style='text-align:right;padding:8px 4px;'>
                                    Variação R$
                                </th><th style='text-align:right;padding:8px 4px;'>Variação %</th>
                            </tr>
                        </thead>
                        <tbody>"
                            & LinhasHTML &
                        "</tbody>
                    </table>
                </div>",
                ""
            )

            & IF(
                Pergunta = 1,
                "<p style='font-size:10px;color:#64748B;'>
                    Resumo de indicadores: as linhas incluem resultados
                    intermediários e não devem ser somadas.
                 </p>",
                ""
            )

            & "<p style='
                font-size:10px;
                color:#64748B;
                margin-top:14px;
            '>
                Fonte: DRE gerencial de caixa · Valores em R$.
                Comparação com o mês anterior, conforme o cenário selecionado.<br>— indica ausência de valor ou percentual sem base. Na diferença em R$, valores ausentes são tratados como zero. As setas indicam aumento (↑), redução (↓) ou estabilidade (→). Nas despesas, indicam a direção do gasto. Verde = efeito favorável; vermelho = desfavorável. Percentual exibido em magnitude: diferença absoluta dividida pelo valor absoluto do mês anterior. Na variação em R$, valores negativos nas saídas indicam aumento do gasto.
             </p>"
    )

VAR AjudaFiltros =
    SWITCH(
        Pergunta,
        7,
        "<h3 style='font-size:16px;'>Como escolho o mês e o ano?</h3>
        <p>Use os filtros <b>ANO</b>, <b>MÊS</b> e <b>CENÁRIO</b> no topo do relatório.</p>
        <ol style='padding-left:22px;line-height:1.9;'>
        <li>Em <b>ANO</b>, selecione o ano desejado.</li>
        <li>Em <b>MÊS</b>, selecione <b>um único mês</b>.</li>
        <li>Em <b>CENÁRIO</b>, escolha os movimentos que deseja analisar.</li>
        <li>Volte à aba <b>Números</b> ou <b>Ação</b> e escolha uma pergunta.</li></ol>
        <div style='background:#F1F5F9;padding:14px;border-radius:10px;margin:16px 0;'><b>Exemplo</b><br>Selecionando <b>2026</b> e <b>Out</b>, a comparação mensal será entre <b>outubro/2026 e setembro/2026</b>, no mesmo cenário. Para janeiro, o mês anterior é dezembro do ano anterior.</div>
        <p><b>Selecionou Todos ou vários meses?</b> Para receber a comparação mensal do assistente, escolha um único mês e ano.</p>
        <p style='font-size:11px;color:#64748B;'>As opções Mensal, Trimestral e Anual organizam a visualização principal. As perguntas de comparação mensal do assistente continuam comparando meses.</p>",
        8,
        "<h3 style='font-size:16px;'>Qual a diferença entre os cenários?</h3>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Realizado</b><p style='margin:6px 0 0;'>Considera os lançamentos com status <b>BAIXADO</b>: recebimentos e pagamentos registrados como realizados.</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Projetado</b><p style='margin:6px 0 0;'>Considera os lançamentos com status <b>EM ABERTO</b> e <b>ATRASO</b>: valores pendentes registrados na base.</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:14px;'><b>Realizado + Projetado</b><p style='margin:6px 0 0;'>Soma os dois cenários: <b>baixados + em aberto + em atraso</b>, dentro do período e dos filtros selecionados.</p></div>
        <p><b>Projetado não significa orçamento:</b> nesta análise, ele vem dos lançamentos pendentes. Pode incluir valores em atraso e não se limita a meses futuros.</p>
        <div style='background:#F1F5F9;padding:14px;border-radius:10px;'><b>Como comparar</b><br>Escolha um mês e mantenha o cenário desejado. O assistente compara o mês selecionado com o anterior usando esse mesmo cenário.</div>
        <p style='font-size:11px;color:#64748B;margin-top:14px;'>Os resultados dependem das datas, dos status e dos demais filtros do relatório. A atualização dos lançamentos pode alterar os valores.</p>",
        BLANK()
    )

VAR AjudaPaginas =
    SWITCH(
        Pergunta,
        9,
        "<div style='text-align:right;margin-bottom:18px;'><span style='display:inline-block;background:#F1F5F9;border:1px solid #D8E2EF;border-radius:12px;padding:10px 14px;'>Como abro e fecho o assistente?</span></div>
        <h3 style='font-size:16px;margin:0 0 14px;'>Acesse a análise quando precisar</h3>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Para abrir</b><p style='margin:6px 0 0;'>Clique no ícone do <b>Assistente</b>, na parte superior do relatório. O painel será exibido sobre a página.</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Para consultar</b><p style='margin:6px 0 0;'>Selecione uma aba e clique em uma das perguntas disponíveis. As respostas de análise consideram o período, o cenário e os filtros aplicados ao relatório.</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Para fechar</b><p style='margin:6px 0 0;'>Clique em <b>Fechar</b>, no canto superior direito do painel, para voltar à visualização do relatório.</p></div>
        <div style='background:#F1F5F9;border-radius:10px;padding:14px;'><b>Para começar outra consulta</b><br>Use <b>Nova análise</b> e escolha outra pergunta. Confira os filtros de ano, mês e cenário antes de analisar os resultados.</div>",
        10,
        "<div style='text-align:right;margin-bottom:18px;'><span style='display:inline-block;background:#F1F5F9;border:1px solid #D8E2EF;border-radius:12px;padding:10px 14px;'>Como alterno entre mensal, trimestral e anual?</span></div>
        <h3 style='font-size:16px;margin:0 0 14px;'>Escolha como visualizar o período</h3>
        <p>Na parte superior do relatório, clique em <b>Mensal</b>, <b>Trimestral</b> ou <b>Anual</b>. Se necessário, feche o assistente para acessar essas opções.</p>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Mensal</b><p style='margin:6px 0 0;'>Exibe os valores por mês. Use para acompanhar variações e identificar em quais meses ocorreram mudanças.</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>Trimestral</b><p style='margin:6px 0 0;'>Agrupa os valores por trimestre: 1T (jan–mar), 2T (abr–jun), 3T (jul–set) e 4T (out–dez).</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:14px;'><b>Anual</b><p style='margin:6px 0 0;'>Agrupa os valores por ano, considerando os filtros selecionados.</p></div>
        <div style='background:#F1F5F9;border-radius:10px;padding:14px;'><b>Atenção ao filtro MÊS</b><br>Para visualizar o ano inteiro ou todos os trimestres, selecione <b>Todos</b> no filtro MÊS e o ano desejado em ANO. Se apenas outubro estiver selecionado, a visão trimestral ou anual continuará restrita a outubro.</div>
        <p style='font-size:11px;color:#64748B;margin-top:14px;'>As perguntas mensais do assistente comparam o mês selecionado com o anterior. Alterar a visualização principal não transforma essas respostas em comparações trimestrais ou anuais.</p>",
        BLANK()
    )

VAR AjudaInterpretacao =
    SWITCH(
        Pergunta,
        11,
        "<div style='text-align:right;margin-bottom:18px;'><span style='display:inline-block;background:#F1F5F9;border:1px solid #D8E2EF;border-radius:12px;padding:10px 14px;'>Como interpretar as setas e os percentuais?</span></div>
        <h3 style='font-size:16px;margin:0 0 14px;'>Direção da mudança e efeito no resultado</h3>
        <p>A comparação usa o mês selecionado e o mês anterior, mantendo o mesmo cenário e os demais filtros da análise.</p>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>O que as setas mostram</b><p style='margin:6px 0 0;'><b>↑</b> aumento · <b>↓</b> redução · <b>→</b> estabilidade.<br>Nos custos e nas despesas, a seta indica a mudança no <b>gasto</b>. Nas receitas e nos resultados, indica a mudança no valor do indicador.</p></div>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:12px;'><b>O que as cores mostram na comparação</b><p style='margin:6px 0 0;'><span style='color:#007A55;'>Verde</span>: efeito favorável, como aumento de receita ou redução de gasto.<br><span style='color:#DC2626;'>Vermelho</span>: efeito desfavorável, como queda de receita ou aumento de gasto.<br>Essas cores indicam o efeito numérico; não explicam, sozinhas, a causa da mudança.</p></div>
        <p><b>Variação em R$:</b> valor do mês selecionado menos o valor do mês anterior. Como as saídas aparecem negativas, uma variação negativa pode representar aumento do gasto.</p>
        <div style='background:#F1F5F9;border-radius:10px;padding:14px;margin-bottom:14px;'><b>Exemplo de aumento de custo</b><br>Mês anterior: −R$ 10.000,00<br>Mês selecionado: −R$ 12.000,00<br>Variação: −R$ 2.000,00<br><span style='color:#DC2626;'>↑ O gasto aumentou 20%.</span></div>
        <p><b>Percentual:</b> no assistente, a magnitude da mudança é calculada pela diferença absoluta dividida pelo valor absoluto do mês anterior, multiplicada por 100. A seta informa a direção.</p>
        <p><b>Quando aparece —:</b> pode indicar ausência de valor ou falta de base para calcular o percentual. Se o mês anterior estiver zerado ou sem valor, o aumento em reais pode ser mostrado, mas não há base percentual válida.</p>
        <p style='font-size:11px;color:#64748B;'>Na diferença em reais, valores ausentes são tratados como zero. Quando um resultado muda de negativo para positivo, ou o contrário, confira também os dois valores em reais para interpretar a mudança.</p>",
        12,
        "<div style='text-align:right;margin-bottom:18px;'><span style='display:inline-block;background:#F1F5F9;border:1px solid #D8E2EF;border-radius:12px;padding:10px 14px;'>O que significa o saldo dos movimentos apresentados?</span></div>
        <h3 style='font-size:16px;margin:0 0 14px;'>O resultado dos movimentos incluídos na análise</h3>
        <p>Essa linha reúne as entradas e saídas consideradas no demonstrativo, dentro do período, do cenário e dos demais filtros selecionados.</p>
        <div style='border:1px solid #D8E2EF;border-radius:10px;padding:14px;margin-bottom:14px;'><b>Como o saldo é composto</b><p style='margin:8px 0 0;'>Resultado após despesas operacionais<br>+ receitas não operacionais<br>+ despesas financeiras<br>+ impostos sobre o resultado<br>+ entradas e saídas de investimentos<br>+ obrigações.</p><p style='font-size:11px;color:#64748B;margin-bottom:0;'>A soma respeita o sinal dos lançamentos: as saídas normalmente entram com valores negativos.</p></div>
        <p><b>Saldo positivo:</b> as entradas consideradas superaram as saídas consideradas no período.</p>
        <p><b>Saldo negativo:</b> as saídas consideradas superaram as entradas consideradas no período.</p>
        <div style='background:#F1F5F9;border-radius:10px;padding:14px;margin:14px 0;'><b>Saldo dos movimentos não é saldo bancário</b><br>Essa linha não acrescenta automaticamente o saldo inicial das contas bancárias. Para conhecer a disponibilidade em banco, é necessário considerar o saldo inicial e todos os movimentos bancários conciliados.</div>
        <p><b>O cenário muda a leitura:</b> em Realizado, são considerados os movimentos baixados; em Projetado, os pendentes; em Realizado + Projetado, ambos.</p>
        <p>Também não deve ser interpretado diretamente como lucro contábil, pois esta análise gerencial de caixa inclui investimentos e obrigações.</p>
        <p style='font-size:11px;color:#64748B;'>Não some todas as linhas do demonstrativo: os resultados intermediários já contêm outras linhas e isso duplicaria valores. A composição depende das classificações e dos lançamentos incluídos nas medidas do relatório.</p>",
        BLANK()
    )

RETURN
    "<div style='
        font-family:Segoe UI;
        color:#2D4868;
        font-size:13px;
        line-height:1.6;
        padding:14px;
        background:#FFFFFF;
        overflow-wrap:anywhere;
    '>"
               & SWITCH(TRUE(), Pergunta IN {11, 12}, AjudaInterpretacao, Pergunta IN {9, 10}, AjudaPaginas, Pergunta IN {7, 8}, AjudaFiltros, Pergunta IN {5, 6}, [38.DRE Ações HTML], Conteudo & [37.DRE Gráfico Aumentos HTML])
    & "</div>"
