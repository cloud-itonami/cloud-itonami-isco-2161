(ns architecture.governor
  "ArchitectureGovernor — the independent safety/traceability layer for
  the ISCO-08 2161 independent building architect actor. Wired as its own
  `:govern` node in `architecture.actor`'s StateGraph, downstream of
  `:advise` — the Advisor has no notion of project provenance, code
  compliance or the architect's professional/legal responsibility, so this
  MUST be a separate system able to reject a proposal (itonami actor
  pattern, per ADR-2607011000 / CLAUDE.md Actors section).

  `check` is a pure function of (request, context, proposal, store) ->
  verdict; it never mutates the store. The StateGraph's `:decide` node
  routes on the verdict:
    :hard? true                → :hold  (irreversible, no write)
    :escalate? true            → :request-approval (interrupt-before)
    otherwise                  → :commit

  HARD invariants (:hard? true, ALWAYS :hold, never overridable):
    1. project provenance  — the request's project must be registered.
    2. no-actuation        — proposal :effect must be :propose.
    3. no-stamping/certification — any attempt to issue a stamped design
       or certify code compliance is a permanent block (that is the
       licensed architect's exclusive professional responsibility).
  ESCALATION invariants (:escalate? true, ALWAYS human sign-off, per the
  README robotics-premise: stamped designs and code compliance certification
  always require human architect sign-off):
    4. :op :flag-code-compliance-issue (always escalates).
    5. design proposals affecting structural/life-safety systems.
    6. low confidence (< `confidence-floor`)."
  (:require [architecture.store :as store]))

(def confidence-floor 0.6)
(def ^:private escalating-ops #{:flag-code-compliance-issue})
(def ^:private life-safety-keywords #{:structural :seismic :fire :egress :accessibility :hvac})

(defn- hard-violations [{:keys [proposal]} project-record]
  (cond-> []
    (nil? project-record)
    (conj {:rule :no-project :detail "未登録 project"})

    (not= :propose (:effect proposal))
    (conj {:rule :no-actuation :detail "effect は :propose のみ許可（直接書込禁止）"})

    (or (= :issue-stamped-design (:op proposal))
        (= :certify-code-compliance (:op proposal)))
    (conj {:rule :no-stamping :detail "stamped designs and code certification are licensed architect's exclusive responsibility"})))

(defn- life-safety-system? [{:keys [scope tags]}]
  (and scope tags (some life-safety-keywords tags)))

(defn check
  "Assess a proposal against `request`/`context`/`proposal` and a
  `store` implementing `architecture.store/Store`. Returns
  `{:ok? bool :violations [...] :confidence n :hard? bool :escalate? bool}`."
  [request context proposal store]
  (let [project-record (store/project store (:project-id request))
        hard (hard-violations {:proposal proposal} project-record)
        hard? (boolean (seq hard))
        conf (or (:confidence proposal) 0.0)
        low? (< conf confidence-floor)
        escalating-op? (contains? escalating-ops (:op proposal))
        life-safety? (and (not hard?) (life-safety-system? proposal))]
    {:ok? (and (not hard?) (not low?) (not escalating-op?) (not life-safety?))
     :violations hard
     :confidence conf
     :hard? hard?
     :escalate? (and (not hard?) (or low? escalating-op? life-safety?))}))
