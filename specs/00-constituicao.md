# Constituição

Regras que nenhuma spec, plano técnico, tarefa ou código pode violar. Se algo entrar em conflito com este documento, este documento vence, e só muda com decisão explícita do dono do projeto.

Referência: subject do ft_transcendence v19 (42 São Paulo).

## Do subject (reprovam o projeto se faltarem)

1. **Um comando.** O projeto inteiro sobe com um único comando, usando containers.
2. **Chrome limpo.** Funciona na versão estável mais recente do Google Chrome, sem nenhum erro ou aviso no console do navegador.
3. **HTTPS em tudo.** Toda comunicação com o backend usa HTTPS.
4. **Validação dupla.** Todo formulário e toda entrada de usuário são validados no frontend e no backend.
5. **Segredos fora do Git.** Credenciais e chaves só em `.env`, ignorado pelo Git, com um `.env.example` versionado de modelo.
6. **Privacidade e Termos.** Páginas de Política de Privacidade e Termos de Uso acessíveis a partir do site, com conteúdo real e relevante para o produto.
7. **Banco bem modelado.** Esquema claro, com relações bem definidas.
8. **Vários usuários ao mesmo tempo.** Vários operadores logados e ativos simultaneamente, com atualizações refletidas em tempo real para todos e ações concorrentes tratadas sem corromper dados.
9. **README conforme o subject.** Primeira linha em itálico no formato exigido, papéis do time, módulos escolhidos com justificativa e pontos, uso de IA.

## Do projeto

10. **Explicável.** Todo código precisa poder ser explicado e alterado ao vivo pelo dono do projeto durante a avaliação. Nada construído "para o futuro": sem abstração, configuração ou funcionalidade que nenhuma spec pediu.
11. **Módulo só conta funcionando.** Nenhum módulo é declarado no README sem funcionar de ponta a ponta numa demonstração.
12. **Dados pessoais (LGPD).** Dados de motoristas são dados pessoais: só acessíveis com login, cada locadora vê apenas os seus, e são apagados automaticamente 5 anos após o encerramento da última locação do motorista.
13. **Dinheiro é rastreável e testado.** Todo valor em dinheiro calculado pelo sistema (cobranças proporcionais, multas, devoluções) tem teste automatizado. Toda alteração de dado de negócio fica registrada no histórico: quem, quando, valor anterior e novo.
14. **Spec antes do código.** Nada é implementado sem estar na spec. Se for preciso mudar, a spec muda primeiro.
15. **Branch e PR.** Tudo entra na `main` por branch e Pull Request, com mensagens de commit claras que descrevem a mudança. Nada é enviado direto para a `main`.
