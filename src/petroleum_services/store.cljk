(ns petroleum-services.store
  "SSoT for the ISIC-08 0910 petroleum services contractor coordination
  actor. Store is a protocol injected into the `petroleum-services.actor`
  StateGraph — `MemStore` is the default, deterministic, zero-dep
  backend; a Datomic/kotoba-server-backed implementation can be
  swapped in without touching the actor or governor (itonami actor
  pattern, per ADR-2607011000 / CLAUDE.md Actors section).

  Domain:

    client    — a registered petroleum services contractor
                (:client-id, :name, :safety-rating)
    well-site — a registered well-site operated by the contractor's
                clients; target of scheduling/logistics coordination
                (:site-id, :operator-id, :location, :risk-level)
    record    — a committed operational record under a client
                (service order intake, crew dispatch, logistics
                coordination, safety incident log) — written ONLY via
                commit-record!, never mutated in place
    ledger    — an append-only audit trail of every proposal/verdict/
                disposition, regardless of outcome (commit, escalate, hold)")

(defprotocol Store
  (client [s client-id])
  (well-site [s site-id])
  (records-of [s client-id])
  (ledger [s])
  (register-client! [s client])
  (register-well-site! [s site])
  (commit-record! [s record])
  (append-ledger! [s fact]))

(defrecord MemStore [a]
  Store
  (client [_ client-id] (get-in @a [:clients client-id]))
  (well-site [_ site-id] (get-in @a [:well-sites site-id]))
  (records-of [_ client-id] (filter #(= client-id (:client-id %)) (:records @a)))
  (ledger [_] (:ledger @a))
  (register-client! [s client]
    (swap! a assoc-in [:clients (:client-id client)] client) s)
  (register-well-site! [s site]
    (swap! a assoc-in [:well-sites (:site-id site)] site) s)
  (commit-record! [s record]
    (swap! a update :records (fnil conj []) record) s)
  (append-ledger! [s fact]
    (swap! a update :ledger (fnil conj []) fact) s))

(defn mem-store
  ([] (mem-store {}))
  ([seed] (->MemStore (atom (merge {:clients {} :well-sites {} :records [] :ledger []} seed)))))
