.. _bandwidth_tor_code:

Bandwidth code in ``tor``
==========================

To calculate descriptors bandwidth
------------------------------------

As stated in :ref:`bandwidth_tor_desc`:

.. code-block:: none

   bandwidth-avg = bandwidtrate = get_effective_bwrate()
                 = min(RelayBandwidthRate, BandwidthRate, MaxAdvertisedBandwidth)
   bandwidth-burst = bandwidthburst = get_effective_bwburst()
                   = min(RelayBandwidthBust, BandwidthBurst)
   bandwidth-observed = bandwidthcapacity = rep_hist_bandwidth_assess()

.. code-block:: none

  advertised bandwidth = router_get_advertised_bandwidth_capped()
                       = min(bandwidtrate, bandwidthcapacity, 10MB/s)

```rep_hist_bandwidth_asses`` that reads from ``bw_array_t``
Example::

    bandwidthcapacity = max?(r 10547200, w 10536960)

In ``/or/router.c``:

.. code-block:: c

    router_perform_bandwidth_test

    router_build_fresh_descriptor

    /* compute ri->bandwidthrate as the min of various options */
    ri->bandwidthrate = get_effective_bwrate(options);
    ri->bandwidthburst = get_effective_bwburst(options);
    ri->bandwidthcapacity = hibernating ? 0 : rep_hist_bandwidth_assess();

    router_dump_router_to_string
      (int) router->bandwidthrate,
      (int) router->bandwidthburst,
      (int) router->bandwidthcapacity,

In ``/or/rephist.c``:

.. code-block:: c

    rep_hist_bandwidth_asses

    commit_max


In ``/or/dirserv.c``:

.. code-block:: c

    set_routerstatus_from_routerinfo

    dirserv_get_credible_bandwidth_kb

measured bandwidth (by bwauths) or the self advertised bandwidth, but not any self measured bandwidth

In ``/or/dirvote.c``:

.. code-block:: c

    networkstatus_compute_consensus

    smartlist_add_asprintf(chunks, "w Bandwidth=%d%s%s\n",
                             rs_out.bandwidth_kb,
                             unmeasured?" Unmeasured=1":"",
                             guardfraction_str ? guardfraction_str : "");

To publish descriptor
-----------------------

In ``router.c``

.. code-block:: c

  /** By which factor bandwidth shifts have to change to be considered large. */
  #define BANDWIDTH_CHANGE_FACTOR 2

  #define MAX_BANDWIDTH_CHANGE_FREQ (3*60*60)
  /** How frequently will we republish our descriptor because of large (factor
   * of 2) shifts in estimated bandwidth? */

  /** Maximum uptime to republish our descriptor because of large shifts in
   * estimated bandwidth. */
  #define MAX_UPTIME_BANDWIDTH_CHANGE (24*60*60)

  /* If the relay uptime is bigger than MAX_UPTIME_BANDWIDTH_CHANGE,
  * the next regularly scheduled descriptor update (18h) will be enough */
  if (get_uptime() > MAX_UPTIME_BANDWIDTH_CHANGE)

To calculate vote and consensus bandwidth
-----------------------------------------

As stated in :ref:`bandwidth_tor_cons`:


.. code-block:: none

  Bandwidth = advertised bandwidth
            = min(bandwidth-avg, bandwidth-observed, 10MB/s) (KB/s)

If 3 or more authorities provide a Measured= keyword::

  Bandwidth = consensus bandwidth * ratio(avg stream, network avg)

In ``dirserv.c``:

.. code-block:: c

    if (format == NS_CONTROL_PORT && rs->has_bandwidth) {
      bw_kb = rs->bandwidth_kb;
    } else {
      tor_assert(desc);
      bw_kb = router_get_advertised_bandwidth_capped(desc) / 1000;
    }
    smartlist_add_asprintf(chunks,
                     "w Bandwidth=%d", bw_kb);

    if (format == NS_V3_VOTE && vrs && vrs->has_measured_bw) {
      smartlist_add_asprintf(chunks,
                       " Measured=%d", vrs->measured_bw_kb);
    }

.. code-block:: c

  STATIC int
  measured_bw_line_apply(measured_bw_line_t *parsed_line,
                         smartlist_t *routerstatuses)
  {
    vote_routerstatus_t *rs = NULL;
    if (!routerstatuses)
      return 0;

    rs = smartlist_bsearch(routerstatuses, parsed_line->node_id,
                           compare_digest_to_vote_routerstatus_entry);

    if (rs) {
      rs->has_measured_bw = 1;
      rs->measured_bw_kb = (uint32_t)parsed_line->bw_kb;
    } else {
      log_info(LD_DIRSERV, "Node ID %s not found in routerstatus list",
               parsed_line->node_hex);
    }

    return rs != NULL;
  }

In ``dirvote.h``

.. code-block:: c

  #define DEFAULT_MAX_UNMEASURED_BW_KB 20

Bandwidth self-test
--------------------

Reasons (https://trac.torproject.org/projects/tor/ticket/22453#comment:20)

First, the reachability checks are performed, which open testing circuits.
When the OR port is found reachable, it updates the descriptor (still no
bandwidth test?).
Then, the bandwidth test is performed, if it's the first time.

In ``mainloop.c``:

.. code-block:: c

  static int
  check_for_reachability_bw_callback(time_t now, const or_options_t *options)
      [...]
      router_do_reachability_checks(1, dirport_reachability_count==0);
      [...]
      reset_bandwidth_test();
      [...]
#define MIN_BANDWIDTH_RECHECK_INTERVAL (12*60*60)
#define MAX_BANDWIDTH_RECHECK_INTERVAL (24*60*60)

In ``circuituse.c``:

It checks there're testing circuits, if there're are, calls ``router_perform_bandwidth_test``

In ``selftest.c``:

Sends drop cells.

.. code-block:: c

  void
  router_perform_bandwidth_test(int num_circs, time_t now)
    [...]
    log_notice(LD_OR,"Performing bandwidth self-test...done.");

Bandwidth observed
-------------------

In ``rephist.c``

.. code-block:: c

  /** How large are the intervals for which we track and report bandwidth use? */
  #define NUM_SECS_BW_SUM_INTERVAL (24*60*60)

  /** How far in the past do we remember and publish bandwidth use? */
  #define NUM_SECS_BW_SUM_IS_VALID (24*60*60)

  #define NUM_TOTALS (NUM_SECS_BW_SUM_IS_VALID/NUM_SECS_BW_SUM_INTERVAL)

  NUM_SECS_ROLLING_MEASURE 10

  rep_hist_bandwidth_assess


  /** Allocate, initialize, and return a new bw_array. */
  static bw_array_t *
  bw_array_new(void)

    b->next_period = start + NUM_SECS_BW_SUM_INTERVAL

  /** Shift the current period of b forward by one. */
  STATIC void
  commit_max(bw_array_t *b)

set from the begining the `next_period` to 1 day, so `commit_max` doesn't happen until then.

