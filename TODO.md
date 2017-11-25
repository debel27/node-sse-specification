# TODO

  - how to handle congestion ? Congestion may occur when a large connection pool needs to be browsed several times in quick succession.
    Idea : add to the SSEService constructor an `opts.congestionAvoidanceStrategy {sse.CongestionAvoidanceStrategy}` (optional).