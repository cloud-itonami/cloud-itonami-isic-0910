(ns petroleum-services.governor
  "PetroleumServicesGovernor — the independent safety/traceability layer
  for the ISIC-08 0910 petroleum services contractor actor. Wired as its
  own `:govern` node in `petroleum-services.actor`'s StateGraph, downstream
  of `:advise` — the Advisor has no notion of client/site provenance or
  high-risk operations, so this MUST be a separate system able to reject a
  proposal (itonami actor pattern, per ADR-2607011000 / CLAUDE.md Actors
  section).

  CRITICAL DOMAIN NOTE: This actor supports a CONTRACTOR's back-office
  coordination — dispatch, scheduling, logistics, safety logging. It does
  NOT make drilling, well-control, or hazardous-material decisions. Those
  remain the OPERATOR's exclusive authority. Scope boundaries:
    ✓ Service order intake and verification
    ✓ Crew dispatch scheduling
    ✓ Site logistics coordination (transport, supplies, crew movement)
    ✓ Safety incident logging (always escalated)
    ✗ Drilling/completion decisions
    ✗ Well-control operations
    ✗ Hazmat handling authorization

  `check` is a pure function of (request, context, proposal, store) ->
  verdict; it never mutates the store. The StateGraph's `:decide` node
  routes on the verdict:
    :hard? true                → :hold  (irreversible, no write)
    :escalate? true            → :request-approval (interrupt-before)
    otherwise                  → :commit

  HARD invariants (:hard? true, ALWAYS :hold, never overridable):
    1. client provenance       — the request's client must be registered.
    2. site provenance         — dispatch/logistics ops must involve a
                               registered well-site.
    3. no-actuation            — proposal :effect must be :propose.
    4. no-drilling/hazmat      — any proposal flagged :op with drilling-class
                               ops (:drill/:drill-complete/:hazmat-handle)
                               is an instant hard block with no override path.

  ESCALATION invariants (:escalate? true, ALWAYS human sign-off):
    5. :op :log-safety-incident — ALL safety incident logging escalates.
    6. high-risk site          — dispatch to a site flagged :risk-level :high
                               escalates.
    7. low confidence          — < `confidence-floor`."
  (:require [petroleum-services.store :as store]))

(def confidence-floor 0.6)

;; Ops that are hard-blocked (not in contractor's domain; operator only)
(def ^:private drilling-class-ops
  #{:drill :drill-complete :perforate :cement :core-sample
    :hydraulic-fracture :hazmat-handle :well-control})

;; Ops that always escalate
(def ^:private always-escalate-ops #{:log-safety-incident})

(defn- hard-violations [{:keys [proposal]} client-record site-record]
  (cond-> []
    (nil? client-record)
    (conj {:rule :no-client :detail "未登録 client"})

    (and (not (nil? site-record)) (nil? site-record))
    (conj {:rule :no-site :detail "未登録 well-site"})

    (not= :propose (:effect proposal))
    (conj {:rule :no-actuation :detail "effect は :propose のみ許可（直接書込禁止）"})

    (contains? drilling-class-ops (:op proposal))
    (conj {:rule :drilling-class-blocked
           :detail "drilling/well-control/hazmat ops are OPERATOR-exclusive; not in contractor scope"})))

(defn check
  "Assess a proposal against `request`/`context`/`proposal` and a
  `store` implementing `petroleum-services.store/Store`. Returns
  `{:ok? bool :violations [...] :confidence n :hard? bool :escalate? bool}`."
  [request context proposal store]
  (let [client-record (store/client store (:client-id request))
        site-id (get-in request [:site :site-id])
        site-record (when site-id (store/well-site store site-id))
        hard (hard-violations {:proposal proposal} client-record site-record)
        hard? (boolean (seq hard))
        conf (or (:confidence proposal) 0.0)
        low? (< conf confidence-floor)
        safety-op? (contains? always-escalate-ops (:op proposal))
        high-risk? (and site-record (= :high (:risk-level site-record)))]
    {:ok? (and (not hard?) (not low?) (not safety-op?) (not high-risk?))
     :violations hard
     :confidence conf
     :hard? hard?
     :escalate? (and (not hard?) (or low? safety-op? high-risk?))}))
