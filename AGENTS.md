# JIFC: Projeto Técnico do Sistema de Campeonato

Este documento descreve a implementação proposta para a plataforma web de gerenciamento do JIFC. Deve orientar decisões de produto e desenvolvimento; não substitui o regulamento oficial de cada edição.

## 1. Objetivo e premissas

Construir um sistema responsivo para preparar e operar campeonatos com várias modalidades, acompanhar partidas e provas, registrar e validar resultados e publicar programação e classificação.

### Direção de interface

Usar como base o protótipo **Linha de jogo** em `index.html`: painel operacional por data, agenda em linhas, resumo de estados, filtros e coluna de atenção/locais. Preservar a hierarquia enxuta e escaneável, com tabela/lista como estrutura principal, e não transformar cada dado em cartão decorativo. Os outros conceitos do protótipo podem inspirar páginas públicas, mas não são a direção principal do painel administrativo.

### Premissas confirmadas

- O campeonato inclui modalidades coletivas, individuais e provas de atletismo.
- A interface inicial prioriza a operação do dia: agenda, partidas em andamento, pendências e locais.
- Horários, participantes, resultados e classificações devem ser consultáveis por modalidade e edição.
- Resultados devem ter estados distintos antes e depois da validação/publicação.
- O conteúdo existente é apenas um protótipo estático. Não há framework, API ou banco de dados configurados neste repositório.

### Premissas em aberto

Não inventar regras oficiais nem considerá-las confirmadas. Antes de ativar a apuração automática, a organização precisa aprovar, por modalidade e edição: formato, categorias, pontuação, períodos/sets, desempates, elegibilidade, desistência/W.O., tentativas, apuração de baterias e fluxo de recursos. O PDF de futsal mencionado anteriormente não está presente no workspace; quando fornecido, verificar versão, escopo e adaptações locais.

“800 m revezamento” requer confirmação da distância total, número de integrantes, trechos e forma de classificação. Também devem ser confirmados o público-alvo, volume de participantes, calendário, locais, regras de inscrição e critérios da classificação geral.

## 2. Escopo funcional

### Dentro do escopo inicial

- Criar e configurar uma edição do campeonato.
- Cadastrar modalidades, categorias/divisões, instituições/equipes, participantes, locais, quadras/mesas/pistas e oficiais.
- Receber ou registrar inscrições e controlar elegibilidade/aprovação.
- Criar agenda manualmente, atribuir local e responsáveis e avisar conflitos de recursos e participantes.
- Acompanhar partidas e provas por data, modalidade, local e estado.
- Lançar resultados em rascunho, validar, publicar e corrigir com histórico.
- Exibir calendário, resultados e classificação geral pública, de acordo com permissões e regras de privacidade.
- Exportar agenda e resultados em CSV e impressão/PDF numa etapa posterior ao fluxo básico.

### Fora do primeiro corte, salvo decisão explícita

- Cronometragem certificada ou integração automática com placares físicos.
- Pareamento suíço de xadrez e geração automática avançada de tabelas.
- Transmissão de vídeo, chat, pagamentos, controle médico ou gestão de alojamento/transporte.
- Modo offline com sincronização concorrente. Planejar operação tolerante a conexão instável; implementar offline somente após validar necessidade e conflitos de sincronização.
- Publicação automática de resultados ainda não aprovados.

## 3. Arquitetura proposta

Como o repositório não possui stack, adotar a seguinte base apenas após aprovação da equipe:

- **Frontend:** React, TypeScript e Vite; componentes HTML semânticos e CSS responsivo. Separar rotas, componentes, domínio e acesso a API.
- **Backend:** Node.js com TypeScript e Fastify, organizado por módulos de domínio e rotas versionadas `/api/v1`.
- **Persistência:** PostgreSQL com Prisma e migrações versionadas.
- **Validação de contrato:** Zod no limite HTTP e tipos compartilhados gerados/organizados em pacote interno, sem duplicar regras de validação.
- **Autenticação:** sessão segura em cookie `HttpOnly`, `Secure` em produção e `SameSite`; autorização no servidor em toda operação protegida. O provedor de identidade ainda deve ser decidido.
- **Atualização ao vivo:** primeira versão pode usar atualização explícita/periódica. Usar Server-Sent Events para placares publicados somente se os requisitos de latência justificarem.
- **Testes:** Vitest para unidade/componentes e Playwright para fluxos ponta a ponta.

Não adicionar bibliotecas ou serviços sem necessidade demonstrada. Se a equipe escolher outro stack, preservar limites de domínio, contratos, controles de acesso e testes definidos neste documento.

### Limites de módulos

1. **Identidade e acesso:** contas, papéis, associações a uma edição e autorização.
2. **Campeonato:** edição, modalidades, categorias, regulamentos e configuração de pontuação.
3. **Inscrições:** instituições, equipes, atletas, composição de elenco, inscrições e elegibilidade.
4. **Agenda e recursos:** locais, instalações, eventos, alocação de quadra/pista/mesa e detecção de conflitos.
5. **Operação esportiva:** partida, prova, súmula, ocorrências e lançamento de resultados por formato.
6. **Validação e publicação:** revisão, aprovação, retificação, trilha de auditoria e projeção pública.
7. **Consulta e comunicação:** agenda pública, classificação, filtros, exportação e notificações futuras.

## 4. Modelo de domínio

Usar identificadores estáveis (UUID), datas em UTC no armazenamento e fuso horário explícito por edição/local na apresentação. Valores de pontuação não devem ser armazenados como texto formatado.

### Entidades centrais

- `Competition`: identidade permanente do campeonato.
- `Edition`: ano/período, estado, fuso horário, datas, políticas e regulamentos versionados.
- `Sport`: modalidade reutilizável, como futsal, basquete, vôlei, handebol, xadrez, tênis de mesa e atletismo.
- `DisciplineEvent`: disputa concreta dentro de uma modalidade e categoria. Exemplos de atletismo incluem 100 m, 200 m, 400 m, 800 m, 1 km, 2 km, 3 km, revezamento, salto em altura, arremesso de peso, arremesso de dardo, salto com vara e lançamento de disco. Os nomes e parâmetros finais vêm do regulamento da edição.
- `Category`: categoria/divisão configurável (idade, gênero, nível ou outra dimensão aprovada); não inferir atributos sensíveis nem aplicar limites sem regra aprovada.
- `Organization`, `Team`, `Person`, `TeamMembership`, `Entry`: instituição, equipe, participante, vínculo de equipe e inscrição numa disputa.
- `Venue`, `Facility`: sede e recurso reservável, como quadra, ginásio, mesa ou pista.
- `OfficialAssignment`: árbitro, anotador ou responsável atribuído a evento/partida.
- `CompetitionEvent`: item agendável, comum a todos os formatos, com modalidade, categoria, data, local, estado e responsáveis.
- `Match`: evento entre lados/equipes ou participantes, com formato de disputa configurado.
- `Heat` e `Attempt`: séries/baterias e tentativas de uma prova, quando aplicável.
- `ResultRevision`: resultado versionado, autor, horário, motivo, estado e aprovador.
- `RuleSetVersion`: snapshot imutável do regulamento/configuração que rege uma edição ou disputa.
- `AuditLog`: registro de ação administrativa relevante, sem guardar segredos ou dados desnecessários.

### Resultados específicos por modalidade

Evitar uma tabela universal que pressuponha dois placares inteiros para todas as disputas. Cada `CompetitionEvent` referencia um tipo de resultado validado no servidor:

- **Esportes coletivos:** lados, placar, período/set quando necessário, vencedor/empate segundo a configuração, ocorrências estruturadas e referência à súmula.
- **Xadrez:** participantes, resultado de cada partida/rodada, cor/posição quando aplicável, ausência e critérios de classificação definidos pelo formato. Não codificar sistema suíço sem requisito e algoritmo aprovados.
- **Tênis de mesa:** individual, duplas ou equipes conforme configuração; guardar parciais por set quando exigido e resultado agregado.
- **Corridas:** atleta/equipe, bateria, posição, tempo com precisão/unidade, situação (concluiu, não largou, não concluiu, desclassificado) e progressão para final conforme regulamento.
- **Saltos e arremessos:** participante, unidade, lista ordenada de tentativas, marca/altura, tentativa válida/nula e critério de classificação configurado.
- **Revezamento:** equipe inscrita e integrantes por trecho segundo a definição aprovada da prova; registrar resultado e ocorrências sem presumir composição/distância.

Persistir a unidade de medida e precisão no tipo/configuração da prova. Guardar a entrada original e o valor normalizado quando necessário para auditoria e ordenação confiável.

### Estados de evento e resultado

O fluxo sugerido para evento é `draft -> scheduled -> ready -> live -> result_pending -> approved -> published`. Permitir `postponed`, `cancelled` e `walkover` somente com permissões e definição explícitas no regulamento. Nem toda modalidade usa todos os estados.

Resultado é uma revisão imutável: criar rascunho, submeter para revisão, aprovar/publicar ou rejeitar com motivo. Correção depois da publicação cria nova revisão, mantém a anterior e registra autor, horário e justificativa. Nunca sobrescrever silenciosamente um resultado publicado.

## 5. Permissões e perfis

Implementar autorização por papel e escopo da edição, no servidor:

- **Administrador do sistema:** provisiona organizações e configurações globais essenciais.
- **Organizador da edição:** configura edição, regulamentos, modalidades, categorias, inscrições, agenda e acessos da edição.
- **Coordenador de modalidade:** opera eventos da modalidade atribuída e encaminha resultados à revisão.
- **Árbitro/anotador:** consulta partidas atribuídas e lança súmula/resultados permitidos; não publica o próprio resultado salvo regra aprovada.
- **Representante de equipe:** envia inscrições e consulta dados da própria equipe; não altera resultados.
- **Público:** consulta apenas conteúdo publicado e campos autorizados.

Aplicar princípio do menor privilégio. Verificar associação à edição e ao recurso em cada requisição; não confiar em IDs ocultos na interface. Ações críticas exigem registro de auditoria.

## 6. UX e estrutura de páginas

### Painel operacional (direção Linha de jogo)

- Cabeçalho com identidade da edição, seletor de data/etapa, navegação, perfil e ação principal conforme permissão.
- Título “Operação do dia” com data/localização inequívocos.
- Resumo textual: eventos de hoje, ao vivo, próximos, pendentes de validação e alertas. Os totais devem vir de consulta real e respeitar filtros.
- Filtros combináveis por modalidade, categoria, local, estado e busca de equipe/participante; filtros devem poder ser removidos com teclado e anunciados claramente.
- Agenda em linhas que mostre horário, participantes/equipes, modalidade, local, estado, placar/marca e ação contextual. No mobile, reordenar conteúdo em blocos compactos mantendo horário, nomes, estado e ação visíveis.
- Área de atenção para conflitos, atraso, dados incompletos e revisão pendente, com link direto para corrigir cada item.
- Painel de locais/recursos com estado e próxima reserva.
- Estados de carregamento, vazio, erro, acesso negado e ausência de conexão; nenhuma tela vazia sem explicação/ação possível.

### Fluxos principais

1. **Configurar edição:** criar edição -> definir fuso e datas -> configurar modalidades/categorias -> registrar regulamento e versões -> atribuir papéis.
2. **Inscrever participantes:** escolher prova/categoria -> informar equipe/participantes -> validar obrigatoriedade e elegibilidade -> submeter -> aprovar/rejeitar com motivo.
3. **Montar agenda:** criar evento -> escolher participantes, local e horário -> consultar conflitos -> confirmar -> publicar agenda.
4. **Operar uma disputa:** abrir evento atribuído -> conferir participantes/local -> iniciar operação conforme permissões -> lançar ocorrências e resultado específico -> salvar rascunho -> submeter.
5. **Revisar/publicar:** comparar resultado com súmula -> aprovar ou devolver com motivo -> publicar resultado e atualizar classificação -> manter histórico.
6. **Corrigir após publicação:** criar revisão com justificativa -> aprovação por papel autorizado -> recalcular projeções afetadas -> registrar auditoria.

## 7. Acessibilidade e design

Meta de conformidade: WCAG 2.2 nível AA, validada por testes automáticos e manuais. Acessibilidade é requisito transversal, não etapa visual final.

- HTML semântico; regiões `header`, `nav`, `main`, `aside` e títulos hierárquicos.
- Teclado completo, foco visível, ordem de foco coerente, abas com setas e Home/End, diálogos com foco contido e devolvido ao acionador.
- Contraste mínimo AA; não comunicar estado apenas por cor. Texto como “Em andamento”, “Aguardando validação” e ícone/texto juntos.
- Rótulos associados a filtros e campos; erros identificados em texto e ligados ao campo; ações destrutivas ou publicação requerem confirmação adequada.
- Atualizações ao vivo não devem anunciar cada segundo de cronômetro; usar região `aria-live="polite"` para mudança relevante de placar/estado.
- Tabelas de dados com cabeçalhos semânticos; em telas estreitas, oferecer composição responsiva legível sem esconder relações entre cabeçalho e valor.
- Suportar zoom de 200%, largura de 320 CSS px sem perda de conteúdo/ação e preferência por movimento reduzido.
- Não depender de imagem remota para informação essencial; fornecer texto alternativo e comportamento visual aceitável sem carregamento externo.

## 8. API proposta

Todos os endpoints administrativos exigem sessão e autorização. Validar entrada e saída; paginar listas; retornar erros consistentes sem dados internos.

- `GET /api/v1/editions/{editionId}/dashboard?date=YYYY-MM-DD&sportId=&venueId=`: resumo e eventos do dia.
- `GET|POST /api/v1/editions/{editionId}/events`: listar/criar eventos.
- `GET|PATCH /api/v1/events/{eventId}`: obter/editar evento conforme estado e permissão.
- `POST /api/v1/events/{eventId}/schedule-check`: verificar conflitos de local, participante/equipe e oficial.
- `POST /api/v1/events/{eventId}/result-revisions`: criar ou submeter revisão de resultado.
- `POST /api/v1/result-revisions/{revisionId}/approve`: aprovar/publicar conforme política; exigir justificativa para retificação.
- `POST /api/v1/result-revisions/{revisionId}/reject`: devolver revisão com motivo.
- `GET /api/v1/public/editions/{editionSlug}/schedule` e `/results`: dados públicos publicados, sem PII desnecessária.
- `GET /api/v1/public/editions/{editionSlug}/standings`: classificação calculada a partir de resultados aprovados/publicados e versão de regras identificada.

Usar controle de concorrência (por exemplo, `updatedAt` ou versão) para evitar que duas pessoas sobrescrevam alterações. Operações que aprovam/publicam e recalculam classificação devem ser transacionais ou idempotentes. Eventos ao vivo, se introduzidos, devem notificar apenas revisões publicadas.

## 9. Regras de agenda e classificação

O verificador de conflitos é consultivo no MVP: apresentar conflito e motivo antes da confirmação, com política de bloqueio configurável por tipo. Verificar intervalos sobrepostos para o mesmo recurso, equipe/atleta e oficial; considerar fuso horário e duração configurada. Alteração de horário/local deve criar evento de auditoria e futuramente notificação.

Classificação não deve embutir uma tabela universal de pontos. Implementar calculador isolado, puro e testável, identificado pela versão de regra da edição. Recalcular apenas a partir de revisões aprovadas/publicadas. Tornar critérios, desempates, pontuação de participação e eventual consolidação entre modalidades dados configuráveis e revisáveis. Exibir a regra aplicada junto da tabela pública.

## 10. Segurança, privacidade e operação

- Tratar dados pessoais conforme LGPD: definir finalidade, base legal, minimização, retenção, atendimento de direitos e transparência antes de coletar dados reais.
- Não publicar data de nascimento completa, documentos, contatos, lista de inscritos ou dados de menores sem necessidade e autorização aplicável. Separar a projeção pública dos dados administrativos.
- Proteger cookies, validar CSRF quando aplicável, escapar conteúdo, parametrizar consultas, limitar tentativas de autenticação e impor rate limits nos endpoints sensíveis.
- Armazenar segredos em ambiente seguro, nunca em frontend ou repositório; exigir TLS em produção e backups criptografados.
- Manter trilha de auditoria para inscrição, mudança de agenda, lançamento/retificação, aprovação, publicação e mudança de permissões.
- Logs sem tokens, documentos ou conteúdo desnecessário; definir monitoramento de erros, backups e procedimento testado de restauração.
- Exibir horário local coerente e registrar timestamps em UTC. Documentar política de retenção e exclusão/anonimização.

## 11. Qualidade e estratégia de testes

### Unidade e domínio

- Transições permitidas e proibidas de estado.
- Validação de formatos de resultado e unidade/precisão.
- Cálculo de classificação/desempate por versões de regras aprovadas.
- Verificação de conflitos de agenda, incluindo limites de horário e fusos.
- Autorização por papel, edição e atribuição.
- Revisão imutável e auditoria de retificação.

### Integração e interface

- Contratos HTTP, persistência relacional, rollback de migração em ambiente de teste e idempotência de publicação.
- Fluxo completo de inscrição -> agenda -> resultado rascunho -> revisão -> publicação -> consulta pública.
- Filtros combinados, estados vazios, erro/retry e concorrência em edição de resultado.
- Teclado sem mouse, foco de diálogo, leitor de tela nos estados centrais, contraste e zoom.
- Playwright em larguras móvel e desktop; evitar overflow horizontal e validar que controles permanecem operáveis.
- Testes de regressão para alterações de resultado e atualização da classificação.

### Critérios de aceite do painel inicial

- Usuário autorizado consegue filtrar agenda por data, modalidade, local e estado sem perder contexto.
- Cada linha comunica evento, horário, participantes, local, estado e ação disponível.
- Pendências e conflitos apontam diretamente para o registro relacionado.
- Resultado rascunho não aparece como publicado; publicação exige autorização e gera auditoria.
- Interface funciona com teclado e em 320 CSS px/zoom 200%, sem estado informado apenas por cor.
- Dados públicos contêm somente campos autorizados e resultados publicados.

## 12. Plano de implementação

1. **Descoberta e decisões:** receber regulamentos e confirmar categorias, formatos, elegibilidade, formatos de resultado, classificação, papéis, volumes, privacidade, fuso e prioridade do MVP. Registrar decisões e responsáveis.
2. **Fundação do projeto:** aprovar stack, configurar lint/format/typecheck/teste, ambientes, migrações, autenticação, autorização e pipeline de integração contínua.
3. **Domínio e administração:** implementar edição, modalidades, categorias, organizações, equipes, participantes, instalações e versões de regra.
4. **Agenda:** implementar eventos, conflitos consultivos, painel Linha de jogo, filtros, estados e navegação responsiva.
5. **Resultados:** construir interfaces específicas para formatos necessários, rascunho, validação, publicação e auditoria.
6. **Consulta pública:** publicar agenda, resultados e classificação calculada exclusivamente com dados aprovados.
7. **Hardening:** concluir auditoria de acessibilidade, segurança e privacidade; testar volume esperado, backups/restauração e operação em conexão instável.

Cada etapa deve terminar com critérios de aceite, revisão do regulamento aplicável e demonstração com dados fictícios antes de importar dados pessoais reais.

## 13. Instruções para agentes que trabalharem neste repositório

- Ler este documento e o código local antes de alterar arquitetura ou regras de domínio.
- Preservar a direção Linha de jogo para operações administrativas, a menos que o responsável pelo produto selecione outra proposta.
- Não inferir regras de modalidades. Solicitar/registar a regra aprovada e adicionar testes para qualquer cálculo automático.
- Manter resultados específicos por formato sem forçar provas de atletismo/xadrez num modelo de placar de equipes.
- Não expor informações administrativas na API pública nem confiar na ocultação de botões como controle de acesso.
- Para mudanças de schema, incluir migração e testes; para mudanças de fluxo, validar teclado e layouts móvel/desktop.
- Não considerar o protótipo de `index.html` backend funcional: seus dados são estáticos e as interações demonstrativas não persistem.
- Fazer mudanças pequenas e focadas; executar os testes/typecheck relevantes e informar explicitamente validações não executadas.

## 14. Referências e rastreabilidade

Fontes oficiais devem ser consultadas na versão vigente para cada edição, e as adaptações locais devem ser registradas separadamente:

- FIFA, Leis do Jogo de Futsal: <https://inside.fifa.com/technical/football-technology/standards/futsal-laws-of-the-game>
- World Athletics, Book of Rules: <https://worldathletics.org/about-iaaf/documents/book-of-rules>
- FIVB, Regras oficiais de vôlei: <https://www.fivb.com/volleyball/the-game/official-volleyball-rules/>
- FIBA, regras de basquete: <https://www.fiba.basketball/documents/official-basketball-rules>
- IHF, documentos e regras de handebol: <https://www.ihf.info/regulations-documents/361>
- FIDE, handbook e regras de competição: <https://handbook.fide.com/>
- ITTF, handbook: <https://www.ittf.com/handbook/>
- LGPD, Lei nº 13.709/2018: <https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm>

Registrar para cada decisão esportiva: modalidade/prova, versão e trecho da fonte, adaptação aprovada, responsável pela aprovação, data de vigência e testes que demonstram o comportamento esperado.