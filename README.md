# Inteligência de Pista e Estratégia na Stock Car (GP de Curvelo)

Pipeline de dados que transforma relatórios de cronometragem em PDF em análises de estratégia para uma equipe de Stock Car. O objetivo é responder, entre as sessões de um fim de semana de corrida, perguntas que a cronometragem oficial não responde: qual é a degradação real de pneu de cada carro, quanto custou cada pit stop, quanto o tráfego custou em pista e se uma parada antecipada teria funcionado.

Projeto construído em torno dos carros 10 (Ricardo Zonta) e 80 (Alfredinho Ibiapina), da Full Time Sports.

## O Desafio

A cronometragem oficial entrega os dados em PDFs e planilhas de formato fixo, que dificultam qualquer cruzamento. Separar ritmo puro de tráfego, isolar o efeito do combustível e calcular o custo real de uma parada exige um tempo de manipulação manual que uma equipe não tem durante o fim de semana.

## Nomenclatura das Sessões

Os arquivos seguem a convenção da categoria: **P** indica Prova (corrida) e **T** indica Treino. Assim, `P1` é a Prova 1 e `T2` é o Treino 2.

## Arquitetura

    data/01_raw/        PDFs e CSVs originais da cronometragem
    data/02_interim/    bases em limpeza
    data/03_processed/  tabelas finais, prontas para os simuladores
    notebooks/          extratores, análises e simuladores (um por arquivo)
    reports/            gráficos e apresentação

## Pipeline

**1. Extração** (`extrator_telemetria.ipynb`, `extrator_pitstops.ipynb`)
Leitura dos CSVs de volta e extração dos pit stops por expressão regular sobre o texto do PDF, usando `pdfplumber`. Conversão dos tempos no formato `m:ss.mmm` para segundos absolutos. A extração informa o total de paradas encontradas e a contagem por carro antes de salvar.

**2. Fusão e engenharia de stints** (`fundir_dados.ipynb`, `engenharia_stints.ipynb`, `engenharia_stints_treino.ipynb`)
Junção das voltas com os pit stops, com verificação de que nenhuma volta foi perdida ou duplicada e de que todas as paradas extraídas encontraram a volta correspondente. Em seguida, reconstrução da estrutura de stints: identificação das voltas de entrada e de saída dos boxes e detecção de anomalias (tráfego, safety car) por desvio em relação à mediana do stint. No treino livre, onde não existe relatório de pit stop, as idas à garagem são inferidas por voltas acima de 120 segundos.

**3. Leitura de pista** (`analise_degradacao_treino.ipynb`, `analise_degradacao_corrida.ipynb`)
Regressão linear (SciPy) do tempo de volta contra a volta dentro do stint, estimando a degradação em segundos por volta e o ritmo base de cada jogo de pneus. As análises de treino e de corrida ficam em arquivos separados. Inclui uma tentativa de isolar o efeito do combustível somando de volta a penalidade de peso (ver Limitações).

**4. Auditoria de tráfego** (`analise_tr_fego.ipynb`)
Comparação de cada volta contra a reta de ritmo do próprio stint, classificando as perdas em ar sujo (de 0,4s a 1,5s) e tráfego pesado (de 1,5s a 5,0s).

**5. Custo de pit stop** (`custo_pitstop.ipynb`)
O custo real de cada parada é a soma dos tempos da volta de entrada e da volta de saída, menos duas vezes o ritmo base do stint. O ritmo base usa a mediana das voltas limpas como referência.

**6. Simuladores de decisão** (`simulador_estrategia.ipynb`, `estrategia_safety_car.ipynb`)
Comparação de cenários de undercut e overcut contra a diferença de tempo de box entre as equipes, e tabela de cenários de safety car estimando quanto do custo de uma parada a bandeira amarela absorve.

**7. Visualização** (`grafico_race_trace.ipynb`, `micro_setores.ipynb`)
Race trace, com o gap acumulado de cada carro contra o ritmo de referência do vencedor, e volta ideal calculada pela soma dos melhores setores.

## Tecnologias

Python 3 · Pandas · NumPy · SciPy · Matplotlib · pdfplumber

## Limitações Metodológicas Conhecidas

**1. A correção de combustível é um deslocamento constante, não uma estimativa.**
Somar `x × 0,015` ao tempo de volta antes da regressão aumenta a inclinação em exatamente 0,015 em todos os stints. A coluna de degradação pura é, portanto, a degradação aparente somada a uma constante, e a segunda regressão não acrescenta informação. A direção física está correta, porque o carro fica mais leve e o cronômetro melhora sozinho, mascarando o desgaste. Mas o valor de 0,015 s/volta foi adotado sem estimação a partir dos dados. Estimá lo exigiria regredir o tempo de volta contra a carga de combustível, ou no mínimo uma análise de sensibilidade.

**2. Stints de três voltas produzem degradações fisicamente impossíveis.**
Na análise de treino, o filtro de no mínimo 3 voltas limpas gera resultados como uma melhora de 3,8 segundos por volta, que não representam desgaste de pneu e sim a recuperação de uma volta lenta (tráfego, saída de garagem, incidente). Dos 50 stints analisados no treino, 17 apresentam degradação acima de 1 segundo por volta em módulo. O `r_value` já é calculado pela regressão e deveria ser usado como filtro de qualidade, junto com um mínimo de 5 voltas limpas.

**3. O simulador de undercut e overcut é antissimétrico por construção.**
A fórmula do overcut é a do undercut com o sinal trocado, de modo que a ferramenta nunca pode indicar que as duas manobras funcionam, ou que as duas falham, pela dinâmica de pista. Apenas a diferença de tempo de box, que é constante, desloca os dois lados. Na realidade as duas manobras têm mecanismos distintos: o undercut depende da volta de saída e do tráfego na reentrada, e o overcut depende da evolução da pista. Além disso, o ritmo do adversário e a penalidade de pneu frio são assumidos, embora o projeto já calcule os tempos de volta de saída que permitiriam estimá los.

**4. A matriz de safety car superestima a economia.**
O modelo assume que a volta de entrada nos boxes acontece em ritmo de bandeira verde, quando sob safety car o carro também roda devagar até a entrada dos boxes. Apenas o trecho interno do pit lane mantém o tempo. Por isso a tabela chega a indicar custo de parada negativo, isto é, uma parada mais barata do que não parar. Na prática, a economia real fica entre 40% e 60% do custo do pit stop.

**5. A detecção de anomalias é unilateral.**
Apenas voltas mais lentas que a mediana do stint são marcadas como anomalia. Voltas anormalmente rápidas, como relargadas ou erros de cronometragem, passam pelo filtro e puxam a regressão para baixo, contribuindo para o problema descrito no item 2.

**6. A auditoria de tráfego é parcialmente circular.**
A reta de referência é ajustada sobre as voltas limpas e em seguida usada para avaliar essas mesmas voltas. Isso torna o resíduo delas pequeno por construção e aperta artificialmente a régua de detecção.
