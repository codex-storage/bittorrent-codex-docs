---
tags:
  - bittorrent
---
#bittorrent 

The [[libtorrent-rasterbar|libtorrent]] library public API can be found at [libtorrent reference](https://www.libtorrent.org/reference.html) page. 

The library provides lots of utility functions for loading/processing of the torrent files, handling magnet links, and low level internal functions of the BitTorrent peer-exchange protocol. Skipping those, the core interface used by clients can be limited to Session control and the related *Settings*, *Torrent Handle* and *Resume Data*.

**Session**

| *Session* types and functions |
| --------------------------- |
| `session_handle` |
| `session_proxy` |
| `session` |
| `session_params` |
| `write_session_params()` |
| `read_session_params()` |
| `write_session_params_buf()` |

**Settings**

| *Settings* types and functions |
| --------------------------- |
| `settings_pack` |
| `setting_by_name()` |
| `name_for_setting()` |
| `default_settings()` |
| `high_performance_seed()` |
| `min_memory_usage()` |
| `generate_fingerprint()` |

**Torrent Handle**

| *Torrent Handle* types and functions |
| --------------------------- |
| `block_info` |
| `partial_piece_info` |
| `torrent_handle` |
| `hash_value()` |

**Resume Data**

| *Resume Data* types and functions |
| --------------------------- |
| `read_resume_data()` |
| `write_resume_data_buf()` |
| `write_resume_data()` |
| `write_torrent_file_buf()` |
| `write_torrent_file()` |
| `write_torrent_flags_t` |

How much effort would it be to provide a similar interface on top of the Codex client. Well, it is a substantial work. Some parameters are related to low level internals that still can be configured by the client (yet, from the Open Source client, only [[qBittorrent]] client exposes some settings from the settings pack), and some other parameters might not be relevant to us, e.g. does related to trackers or low level internals of the peer-exchange protocol like for instance `unchoke_interval`.

The number of settings available is overwhelmingly big - making one-to-one analysis does not make sense in my opinion - upon deciding which client we would like to support (if any), only then we should evaluate the details of the required interface.

The approach that could be considered is to first decide which protocol extensions are potentially attractive to Codex starting with the items discussed in [[What BitTorrent has that Codex does not]]. Only after knowing which of those extensions we want to support, it makes sense to focus on related technicalities in the *settings pack* and the *session* management. After identifying the settings that are relevant to Codex, we will need to make sure that the user interface adjusts itself accordingly, when Codex integration is enabled.

To wrap up this section, here are the screenshots of the list of parameters from the [[qBittorrent]] client (the one with the most complete set of options):

![[Pasted image 20241110195407.png]]

![[Pasted image 20241110195544.png]]

![[Pasted image 20241110200105.png]]

![[Pasted image 20241110200233.png]]

![[Pasted image 20241110200313.png]]
### Alerts

BitTorrent has quite extensive set of `Alerts`:

![[Pasted image 20241105084831.png]]

### Python Bindings

I also shortly checked if the Python bindings provided by the library are complete. After building [[libtorrent Python bindings]], we can extract the published public API from Python like this:

```bash
$ python
Python 3.12.3 (main, Sep 11 2024, 14:17:37) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import libtorrent
>>> print(dir(libtorrent))
['__doc__', '__file__', '__loader__', '__name__', '__package__', '__path__', '__spec__', '__version__', 'add_files', 'add_magnet_uri', ...
```

The Python bindings look pretty complete and most of the core functions have direct equivalents in Python. Separating *core functions* from *type definitions* and *alerts*, we get:

**86 core functions**

| function name               |
| --------------------------- |
| `add_files`                 |
| `add_magnet_uri`            |
| `add_torrent_params`        |
| `announce_entry`            |
| `bdecode`                   |
| `bdecode_category`          |
| `bencode`                   |
| `client_fingerprint`        |
| `create_smart_ban_plugin`   |
| `create_torrent`            |
| `create_ut_metadata_plugin` |
| `create_ut_pex_plugin`      |
| `default_settings`          |
| `dht_lookup`                |
| `dht_settings`              |
| `dht_state`                 |
| `enc_level`                 |
| `enc_policy`                |
| `error_category`            |
| `error_code`                |
| `file_entry`                |
| `file_open_mode`            |
| `file_slice`                |
| `file_storage`              |
| `find_metric_idx`           |
| `fingerprint`               |
| `generate_fingerprint`      |
| `generic_category`          |
| `get_bdecode_category`      |
| `get_http_category`         |
| `get_i2p_category`          |
| `get_libtorrent_category`   |
| `get_socks_category`        |
| `get_upnp_category`         |
| `high_performance_seed`     |
| `http_category`             |
| `i2p_category`              |
| `identify_client`           |
| `ip_filter`                 |
| `kind`                      |
| `libtorrent_category`       |
| `load_torrent_buffer`       |
| `load_torrent_file`         |
| `load_torrent_parsed`       |
| `make_magnet_uri`           |
| `min_memory_usage`          |
| `open_file_state`           |
| `operation_name`            |
| `parse_magnet_uri`          |
| `parse_magnet_uri_dict`     |
| `pe_settings`               |
| `peer_class_type_filter`    |
| `peer_id`                   |
| `portmap_protocol`          |
| `portmap_transport`         |
| `protocol_type`             |
| `protocol_version`          |
| `read_resume_data`          |
| `read_session_params`       |
| `session`                   |
| `session_params`            |
| `session_stats_metrics`     |
| `session_status`            |
| `set_piece_hashes`          |
| `sha1_hash`                 |
| `sha256_hash`               |
| `socks_category`            |
| `stats_channel`             |
| `stats_metric`              |
| `system_category`           |
| `torrent_flags`             |
| `torrent_handle`            |
| `torrent_info`              |
| `torrent_status`            |
| `tracker_source`            |
| `upnp_category`             |
| `version`                   |
| `version_major`             |
| `version_minor`             |
| `write_flags`               |
| `write_resume_data`         |
| `write_resume_data_buf`     |
| `write_session_params`      |
| `write_session_params_buf`  |
| `write_torrent_file`        |
| `write_torrent_file_buf`    |

32 type definitions:

| type name |
| ------------- |
| `add_piece_flags_t` |
| `add_torrent_params_flags_t` |
| `bandwidth_mixed_algo_t` |
| `choking_algorithm_t` |
| `create_torrent_flags_t` |
| `deadline_flags_t` |
| `deprecated_move_flags_t` |
| `event_t` |
| `file_flags_t` |
| `file_progress_flags_t` |
| `info_hash_t` |
| `io_buffer_mode_t` |
| `listen_on_flags_t` |
| `metric_type_t` |
| `mmap_write_mode_t` |
| `move_flags_t` |
| `operation_t` |
| `options_t` |
| `pause_flags_t` |
| `peer_class_type_filter_socket_type_t` |
| `performance_warning_t` |
| `proxy_type_t` |
| `reannounce_flags_t` |
| `reason_t` |
| `save_resume_flags_t` |
| `save_state_flags_t` |
| `seed_choking_algorithm_t` |
| `session_flags_t` |
| `socket_type_t` |
| `status_flags_t` |
| `storage_mode_t` |
| `suggest_mode_t` |

**99 Alerts**

| alert type name |
| ------------- |
| `alert` |
| `add_torrent_alert` |
| `alert_category` |
| `alerts_dropped_alert` |
| `anonymous_mode_alert` |
| `block_downloading_alert` |
| `block_finished_alert` |
| `block_timeout_alert` |
| `block_uploaded_alert` |
| `cache_flushed_alert` |
| `dht_announce_alert` |
| `dht_bootstrap_alert` |
| `dht_get_peers_alert` |
| `dht_get_peers_reply_alert` |
| `dht_immutable_item_alert` |
| `dht_live_nodes_alert` |
| `dht_log_alert` |
| `dht_mutable_item_alert` |
| `dht_outgoing_get_peers_alert` |
| `dht_pkt_alert` |
| `dht_put_alert` |
| `dht_reply_alert` |
| `dht_sample_infohashes_alert` |
| `dht_stats_alert` |
| `external_ip_alert` |
| `fastresume_rejected_alert` |
| `file_completed_alert` |
| `file_error_alert` |
| `file_prio_alert` |
| `file_progress_alert` |
| `file_rename_failed_alert` |
| `file_renamed_alert` |
| `hash_failed_alert` |
| `i2p_alert` |
| `incoming_connection_alert` |
| `invalid_request_alert` |
| `listen_failed_alert` |
| `listen_failed_alert_socket_type_t` |
| `listen_succeeded_alert` |
| `listen_succeeded_alert_socket_type_t` |
| `log_alert` |
| `lsd_error_alert` |
| `metadata_failed_alert` |
| `metadata_received_alert` |
| `oversized_file_alert` |
| `peer_alert` |
| `peer_ban_alert` |
| `peer_blocked_alert` |
| `peer_connect_alert` |
| `peer_disconnected_alert` |
| `peer_error_alert` |
| `peer_info_alert` |
| `peer_log_alert` |
| `peer_snubbed_alert` |
| `peer_unsnubbed_alert` |
| `performance_alert` |
| `picker_log_alert` |
| `piece_availability_alert` |
| `piece_finished_alert` |
| `piece_info_alert` |
| `portmap_alert` |
| `portmap_error_alert` |
| `portmap_log_alert` |
| `read_piece_alert` |
| `request_dropped_alert` |
| `save_resume_data_alert` |
| `save_resume_data_failed_alert` |
| `scrape_failed_alert` |
| `scrape_reply_alert` |
| `session_stats_alert` |
| `session_stats_header_alert` |
| `socks5_alert` |
| `state_changed_alert` |
| `state_update_alert` |
| `stats_alert` |
| `storage_moved_alert` |
| `storage_moved_failed_alert` |
| `torrent_added_alert` |
| `torrent_alert` |
| `torrent_checked_alert` |
| `torrent_conflict_alert` |
| `torrent_delete_failed_alert` |
| `torrent_deleted_alert` |
| `torrent_error_alert` |
| `torrent_finished_alert` |
| `torrent_log_alert` |
| `torrent_need_cert_alert` |
| `torrent_paused_alert` |
| `torrent_removed_alert` |
| `torrent_resumed_alert` |
| `tracker_alert` |
| `tracker_announce_alert` |
| `tracker_error_alert` |
| `tracker_list_alert` |
| `tracker_reply_alert` |
| `tracker_warning_alert` |
| `udp_error_alert` |
| `unwanted_block_alert` |
| `url_seed_alert` |

