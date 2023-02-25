---
draft: true
---

# serveHttpKVAPI

store.proposeC -> raft.node.propc

- put
  - store.Propose
    - s.proposeC <- buf.String()
      - prop, ok := <-rc.proposeC
        - rc.node.Propose(context.TODO(), []byte(prop))
          - stepWait
            - n.propc <- pm
              - pm := <-propc:
                - r.Step(m)
                  - r.step(r, m)
                    - r.appendEntry(m.Entries...)
                    - r.bcastAppend()
                  - newPipelineHandler
                - h.r.Process(context.TODO(), m)
              - Raftnode.process
            - rc.node.Step(ctx, m)
          - n.recvc <- m:
        - r.Step(m)
      - n.rn.HasReady()
    - readyc <- rd:
  - rd := <-rc.node.Ready()

