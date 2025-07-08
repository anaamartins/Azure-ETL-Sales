# Azure-ETL-Sales

---PT---

###### Projeto de ETL com Azure Data Factory – Vendas e Indicadores Macroeconómicos
Projeto de ETL desenvolvido em Azure Data Factory no contexto da pós-graduação em Data Science & Business Analytics. O foco foi a construção de um pipeline de dados robusto para análise de vendas de um grupo empresarial e indicadores macroeconómicos em diversos países no período de 2022 a 2024.

##### Objetivo
Construir um modelo analítico completo a partir de múltiplas fontes de dados, integrando ficheiros exportados do ERP SAP, base de dados interna da empresa e indicadores macroeconómicos obtidos de fontes institucionais (PIB, câmbio, inflação, desemprego, petróleo). O pipeline processa, modela e disponibiliza os dados para consulta e visualização através de SQL e Power BI.

##### Tecnologias

    1. Azure Data Factory (ADF) - orquestração de pipelines e transformação

    2. Azure Data Lake Storage Gen2 (ADLS) - armazenamento

    3. Azure Data Studio - integração e consulta

    Formatos: .xlsx, .parquet

##### Arquitetura do Projeto

A estrutura no ADLS foi organizado em três pastas de um _container_:

    **Source** – ficheiros originais do SAP, base de dados interna e fontes institucionais. Armazena todos os ficheiros brutos
    **Raw** – dados convertidos para Parquet, para uma melhor performance e otimização de armazenamento, sem transformações
    **Processed** – recebe os dados já transformados após o tratamento realizado pelos _Data Flows_

Além disso, os dados tratados foram carregados para uma base relacional em **Azure Data Studio**.

O modelo é composto pelas seguintes tabelas dimensão: Cliente, Empresa, País, Calendário, Produto, Projeto, Tipo de Venda e Calendário, que fornecem contexto aos dados de vendas. E pelas tabelas facto - FactVendas e FactMacros - que registam métricas e transações.

Relativamente às tabelas facto é importante salientar que a FactVendas é uma _transactional fact table_ que regista as vendas adjudicadas, e a FactMacros, uma _periodic snapshot table_ que armanena indicadores macroeconómicos em momentos especificos no tempo.

##### Processo de ETL
Baseou-se na recolha de dados provenientes de múltiplas fontes:
- SAP (clientes, empresas, produtos, projeto, taxas de câmbio)
- dados de vendas provenientes de uma base de dados interna do grupo empresarial
- dados dos indicadores macroeconómicos obtidos em sites institucionais de cada país onde ocorreram vendas durante o periodo em análise
  

Para cada uma das fontes de dados configurei _datasets_ no Azure Data Factory. Em seguida, desenvolvi um pipeline que copia e converte os ficheiros do Source (xlsx) e guarda-os na pasta raw em Parquet. Em seguida foram criados _Data Flows_ onde foram aplicadas as transformações necessárias, tanto para as tabelas dimensão como para as tabelas facto. Os principais tratamentos foram os seguintes:

  ###### Tabelas Dimensão

  1. DimCliente
     
     - Seleção das colunas relevantes
     - Tratamento da coluna Cidade: split para remover na coluna Cidade registos que continham Cidade e País e formatação de registos que estavam com texto em Caps Lock de forma a garantir a uniformização dos dados
    
       ![image](https://github.com/user-attachments/assets/5e542b2a-4127-455e-9ad2-d8a259f48d21)

       
  2. DimEmpresa

     - Carregamento direto do ficheiro Parquet (previamente convertido do Excel), sem necessidade de transformações adicionais

        ![image](https://github.com/user-attachments/assets/e37e2ca9-ea18-49f5-b644-211a033837f6)

  3. DimPais

     - Padronização da coluna Pais, formatámos registos em Caps Lock
     - Criação de coluna Mercado (“Nacional” ou “Internacional”) para permitir agrupamentos

  4. DimProduto

     - Seleção das colunas essenciais
     - Criação de chave única que combine produto + empresa, já que um mesmo produto pode ter sido expandido a várias empresas.
     - Padronização dos textos de algumas colunas que tinham textos em caps lock).
  
  5. DimProjeto

     - Seleção das colunas essenciais.
     - Padronização das colunas com registos em Caps Lock.
     - Criação da coluna da Linha de Negócio para que nas análises consigamos perceber a que linha pertencem os projetos vendidos.
     - Formatação da coluna Status para termos maior clareza sobre os status actuais dos projetos

  6. DimTipoVenda

     - Apenas seleção das colunas necessárias, sem transformações adicionais.
    

  ###### Tabelas Facto

  1. FactMacros – integrei diferentes fontes de indicadores macroeconómicos.
     Integração dos seguintes indicadores:
     - Desemprego, inflação, PIB (base anual).
     - Taxa de câmbio e preço do petróleo (base mensal).
       
     **Desafios de granularidade e ETL**:
     - Petróleo: valor no dia 1 de cada mês, assumimos no nosso ETL que seria o preço praticado no **último dia do mês anterior**.
     - Câmbio: valores no último dia útil de cada mês.
     - Criámos uma lista de todos meses, e associámos os indicadores anuais ao último dia de cada mês.
     - Em cada join filtrámos pelo idPais (usando expressão CASE para atribuir país correto).
     - Para o câmbio, definimos que, em países com que a empresa opera com a mesma moeda (Portugal, Malta e Espanha), o valor seria fixado como 1; nos outros, usamos a taxa específica.
     - Seleção final de colunas com as chaves (iddata, idpais) e valores macroeconómicos
    
  3. FactVendas

     - Seleção das colunas principais de vendas.
     - Criação de idProduto concatenando a organização de vendas + código de produto (garante registo único mesmo se o produto pertencer a várias empresas ou diferentes hierarquias).
     - Formatação de datas e valores (conversão de strings para datetime, e formatação das colunas ValorVenda e ValorMargem para valores decimais).
     - Atualização da coluna ValorMargem: quando o tipo de venda é “1ª manutenção” ou “manutenção”, define-se que a margem é igual à faturação, pois não há custos associados.
     - Posteriormente foram identificados casos onde a margem foi incorretamente inserida como sendo o valor de venda, multiplicado por -1 o que levou a existência de custos incorretos. Para solucionar este desafio a coluna Margem foi transformada em ABS o que assegura que todos os valores da margem de vendas são positivos. Assim nos casos onde a margem foi igual à venda o valor do custo aparece corretamente. Esta decisão foi validada com base na natureza das ocorrências, maioritariamente associadas a licenças próprias, onde efectivamente não existe custo associado, justificando margens iguais ao valor da venda.


Após a construção e transformação das tabelas dimensão e facto, foram desenvolvidos dois pipelines no Azure Data Factory: um dedicado ao carregamento das tabelas dimensão e outro ao das tabelas facto. Cada pipeline tem como destino a pasta processed, localizada num container no Azure Data Lake, onde os ficheiros são gravados em formato Parquet, garantindo eficiência no armazenamento e leitura. Paralelamente, os dados processados são também carregados para o Azure Data Studio, permitindo que as equipas de análise e desenvolvimento realizem consultas, validação de dados e análises complementares de forma mais flexível e integrada. Esta abordagem híbrida — com armazenamento em data lake e exposição em SQL — assegura desempenho, rastreabilidade e acessibilidade.
     
