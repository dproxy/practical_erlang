# Live Coding. Выбор мастера с помощью Raft Consensus Algorithm.

Известно, что эрланг предназначен для разработки распределенных
систем. Это сложная область программирования, и о ней можно много
говорить.  Но зачем говорить, когда можно просто взять, и сделать
что-нибудь интересное?

Я проведу сессию программирования на эрланг и реализую механизм выбора
мастера для кластера из эрланговских нод с помощью Raft Consensus
Algorithm.


## Обзор Raft

Raft — алгоритм для решения задач консенсуса в сети надёжных вычислений.

Есть кластер, состоящий из нескольких узлов.
Все узлы содержат копию одного и того же состояния (данные и бизнес-логика).

Для любого кластера характерно:
- часть узлов может падать (и возвращаться);
- может теряться связь между узлами;
- могут добавляться новые узлы в кластер.

При всем этом нужно как-то гарантировать, что состояния на всех узлах идентичны.
(Это и называется консенсус).

Raft предлагает способ, как этого можно добиться.

Это алгоритм общего назначения,
на основе которого можно построить разные прикладные системы.

Raft описан в работе:
"In Search of an Understandable Consensus Algorithm"
Diego Ongaro and John Ousterhout
Stanford University
http://ramcloud.stanford.edu/raft.pdf

https://raft.github.io/
Интерактивная модель
Ссылки на статьи по теме
Ссылки на реализации


## Основы Raft

При работе кластера в обычном режиме один узел выполняет роль Leader,
все остальные узлы роль Follower.

Leader принимает все запросы от клиентов, применяет их к своему состоянию,
и рассылает эти запросы всем остальным узлам, чтобы они тоже применили их
к своему состоянию.

Follower-узлы не взаимодействуют с клиентами и друг с другом, а только
получают запросы от Leader.

img:cluster

Leader может упасть или потерять связь с остальными узлами.
Тогда оставшиеся узлы должны выбрать нового лидера и вернуться в штатный режим работы.

Существуют промежутки времени, когда в кластере нет лидера. И в это время
кластер не может обслуживать запросы клиента (или работает только на чтение).

Период работы от одного выбора лидера до следующего выбора называется Term.
Каждый Term имеет свой номер.

img:term

Каждый запрос от клиента в терминах Raft называется Log, и каждый Log тоже имеет свой id.
Term и log_id важны для поддержания целостности.

Алгоритм Raft делится на отдельные задачи:
- Выбор лидера
- Репликация логов
- Изменение размеров кластера


## Выбор лидера

В каждый момент времени узел может быть в одном из 3-х состояний:
Leader, Follower или Candidate.

Узел стартует в состоянии Follower и ждет запросы от лидера.
Если в течение определенного времени узел не получает запросов,
то он считает, что лидера нет, переходит в состояние Candidat,
и начинает процедуру выбора лидера.

Candidat увеличивает Term на единицу, голосует сам за себя,
и рассылает все остальным узлам в кластере запрос Request Vote.

Follower-узлы, получив такой запрос, голосуют за кандата.
При этом они проверяют Term, и в каждом Term голосуют только один раз,
за первого, от кого придет запрос.

Candidat какое-то время ждет ответы на свои запросы.
Тут могут случиться 3 варианта:
- кандидат получит голоса от большинства узлов в кластере, и станет лидером;
- другой узел станет лидером раньше, тогда кандидат перейдет в состояние Follower;
- никто из кандидатов не получит большинства голосов, тогда выбор лидера запускается заново.

Став лидером, узел сразу рассылает всем остальным узлам в кластере запрос Append Entries.
Этот запрос выполняет 2 роли:
- копирует лог от лидера на все остальные узлы;
- сообщает всем узлам, кто стал лидером в результате голосования.

img:states

Все запросы содержат Term. Все узлы, получив запрос Request Vote или Append Entries
сравнивают Term в запросе со своим собственным. Если Term в запросе меньше, значит
отправитель не актуален, и запрос игнорируется. Если Term в запросе больше,
значит данный узел не актуален. Он обновляет свой Term и переключается в состояние Follower.


## Репликация логов

TODO

Лидер полностью отвечает за правильную репликацию протоколов.

2 фазы: добавление в очередь и коммит

лидер сохраняет лог в своей очереди
рассылает его всем follower
получает от них подтверждение, что они добавили лог в свои очереди
коммитит лог
со следующим append_log идет инфа, какой id закомичен
followers тоже комитят лог

Если внешний клиент подключается к кластеру через обычную ноду, то все его запросы перенаправляются лидеру


## Изменение размеров кластера
TODO
Raft позволяет легко менять конфигурацию кластера, не останавливая работы: добавлять или удалять ноды.

Тут не понятно, надо разобраться и объяснить


==== END ====

## Псевдокод

Если обычная нода долго не получает сообщений от лидера, то она переходит в состояние «кандидат»
и посылает другим нодам запрос на голосование.

Другие ноды голосуют за того кандидата, от которого они получили первый запрос.

Если кандидат получает сообщение от лидера, то он снимает свою кандидатуру и возвращается в обычное состояние.

Если кандидат получает большинство голосов, то он становится лидером.

Если же он не получил большинства (это случай, когда на кластере возникли сразу несколько кандидатов и голоса разделились),
то кандидат ждёт случайное время и инициирует новую процедуру госования.

Процедура голосования повторяется, пока не будет выбран лидер.


basic consensus algorithm requires only
**two types of RPCs**:

RequestVote - initiated by candidates during elections
Append-Entries - initiated by leaders to replicate log entries and to provide a form of heartbeat

Additionally: third RPC for transferring snapshots between servers

Servers retry RPCs if they do not receive a response in a timely manner

When servers start up, they begin as followers
A server remains in follower state as long as it receives valid RPCs from a leader or candidate

Leaders send periodic heartbeats to all followers
(AppendEntries RPCs that carry no log entries)

If a follower receives no communication over a period of time
called the **election timeout**, then it assumes there is no viable leader
and begins an election to choose a new leader.

To begin an election, a follower increments its current term
and transitions to candidate state.

It then votes for itself
and issues RequestVote RPCs in parallel to each of the other servers in the cluster.

A candidate continues in this state until one of three things happens:
- it wins the election,
- another server establishes itself as leader
- a period of time goes by with no winner.


**wins the election**

it receives votes from a majority of the servers in the full cluster for the same term.

Each server will vote for at most one candidate in a given term,
on a first-come-first-served basis

Once a candidate wins an election, it becomes leader.
It then sends heartbeat messages to all of the other servers
to establish its authority and prevent new elections.


**another server establishes itself as leader**

While waiting for votes, a candidate may receive an AppendEntries RPC
from another server claiming to be leader.

If the leader’s term >= candidate’s current term,
candidate recognizes the leader
and returns to follower state.

If the term in the RPC < candidate’s current term,
then the candidate rejects the RPC and continues in candidate state.


**a period of time goes by with no winner**

if many followers become candidates at the same time,
votes could be split so that no candidate obtains a majority.

each candidate will time out
and start a new election by incrementing its term
and initiating another round of Request-Vote RPCs.

Raft uses randomized election timeouts to ensure that
split votes are rare and that they are resolved quickly.

election timeouts are
chosen randomly from a fixed interval (e.g., 150–300ms)

in most cases only a single server will time out
it wins the election and sends heartbeats before any other servers time out