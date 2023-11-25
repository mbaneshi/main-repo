[ChatGPT](https://chat.openai.com/c/465190ac-43c5-4ab7-a0ac-1f1f8cfca260)

<details>
  <summary>

    Source Code

  </summary>
  
```cpp

#pragma once
#include "common/refcnt.hpp"
#include "common/refint.h"
#include "vm/cells.h"
#include "vm/cellslice.h"
#include "vm/dict.h"
#include "vm/boc.h"
#include <ostream>
#include "tl/tlblib.hpp"
#include "td/utils/bits.h"
#include "ton/ton-types.h"
#include "block/block.h"
#include "block/mc-config.h"

namespace block {
using td::Ref;
using LtCellRef = std::pair<ton::LogicalTime, Ref<vm::Cell>>;

struct Account;

namespace transaction {
struct Transaction;
}  // namespace transaction

struct CollatorError {
  std::string msg;
  CollatorError(std::string _msg) : msg(_msg) {
  }
  CollatorError(const char* _msg) : msg(_msg) {
  }
  std::string get_msg() const {
    return msg;
  }
};

static inline bool operator<(const LtCellRef& a, const LtCellRef& b) {
  return a.first < b.first;
}

struct LtCellCompare {
  bool operator()(const LtCellRef& a, const LtCellRef& b) {
    return a.first < b.first;
  }
};

struct NewOutMsg {
  ton::LogicalTime lt;
  Ref<vm::Cell> msg;
  Ref<vm::Cell> trans;
  NewOutMsg(ton::LogicalTime _lt, Ref<vm::Cell> _msg, Ref<vm::Cell> _trans)
      : lt(_lt), msg(std::move(_msg)), trans(std::move(_trans)) {
  }
  bool operator<(const NewOutMsg& other) const& {
    return lt < other.lt || (lt == other.lt && msg->get_hash() < other.msg->get_hash());
  }
  bool operator>(const NewOutMsg& other) const& {
    return lt > other.lt || (lt == other.lt && other.msg->get_hash() < msg->get_hash());
  }
};

struct StoragePhaseConfig {
  const std::vector<block::StoragePrices>* pricing{nullptr};
  td::RefInt256 freeze_due_limit;
  td::RefInt256 delete_due_limit;
  StoragePhaseConfig() = default;
  StoragePhaseConfig(const std::vector<block::StoragePrices>* _pricing, td::RefInt256 freeze_limit = {},
                     td::RefInt256 delete_limit = {})
      : pricing(_pricing), freeze_due_limit(freeze_limit), delete_due_limit(delete_limit) {
  }
};

struct StoragePhase {
  td::RefInt256 fees_collected;
  td::RefInt256 fees_due;
  ton::UnixTime last_paid_updated;
  bool frozen{false};
  bool deleted{false};
  bool is_special{false};
};

struct ComputePhaseConfig {
  td::uint64 gas_price;
  td::uint64 gas_limit;
  td::uint64 special_gas_limit;
  td::uint64 gas_credit;
  td::uint64 flat_gas_limit = 0;
  td::uint64 flat_gas_price = 0;
  static constexpr td::uint64 gas_infty = (1ULL << 63) - 1;
  td::RefInt256 gas_price256;
  td::RefInt256 max_gas_threshold;
  std::unique_ptr<vm::Dictionary> libraries;
  Ref<vm::Cell> global_config;
  td::BitArray<256> block_rand_seed;
  bool ignore_chksig{false};
  bool with_vm_log{false};
  td::uint16 max_vm_data_depth = 512;
  std::unique_ptr<vm::Dictionary> suspended_addresses;
  int vm_log_verbosity = 0;
  ComputePhaseConfig(td::uint64 _gas_price = 0, td::uint64 _gas_limit = 0, td::uint64 _gas_credit = 0)
      : gas_price(_gas_price), gas_limit(_gas_limit), special_gas_limit(_gas_limit), gas_credit(_gas_credit) {
    compute_threshold();
  }
  ComputePhaseConfig(td::uint64 _gas_price, td::uint64 _gas_limit, td::uint64 _spec_gas_limit, td::uint64 _gas_credit)
      : gas_price(_gas_price), gas_limit(_gas_limit), special_gas_limit(_spec_gas_limit), gas_credit(_gas_credit) {
    compute_threshold();
  }
  void compute_threshold();
  td::uint64 gas_bought_for(td::RefInt256 nanograms) const;
  td::RefInt256 compute_gas_price(td::uint64 gas_used) const;
  void set_gas_price(td::uint64 _gas_price) {
    gas_price = _gas_price;
    compute_threshold();
  }
  Ref<vm::Cell> lookup_library(td::ConstBitPtr key) const;
  Ref<vm::Cell> lookup_library(const td::Bits256& key) const {
    return lookup_library(key.bits());
  }
  Ref<vm::Cell> get_lib_root() const {
    return libraries ? libraries->get_root_cell() : Ref<vm::Cell>{};
  }
  bool parse_GasLimitsPrices(Ref<vm::CellSlice> cs, td::RefInt256& freeze_due_limit, td::RefInt256& delete_due_limit);
  bool parse_GasLimitsPrices(Ref<vm::Cell> cell, td::RefInt256& freeze_due_limit, td::RefInt256& delete_due_limit);
  bool is_address_suspended(ton::WorkchainId wc, td::Bits256 addr) const;

 private:
  bool parse_GasLimitsPrices_internal(Ref<vm::CellSlice> cs, td::RefInt256& freeze_due_limit,
                                      td::RefInt256& delete_due_limit, td::uint64 flat_gas_limit = 0,
                                      td::uint64 flat_gas_price = 0);
};

struct ActionPhaseConfig {
  int max_actions{255};
  int bounce_msg_body{0};  // usually 0 or 256 bits
  MsgPrices fwd_std;
  MsgPrices fwd_mc;  // from/to masterchain
  SizeLimitsConfig size_limits;
  const WorkchainSet* workchains{nullptr};
  td::optional<td::Bits256> mc_blackhole_addr;
  const MsgPrices& fetch_msg_prices(bool is_masterchain) const {
    return is_masterchain ? fwd_mc : fwd_std;
  }
};

struct CreditPhase {
  td::RefInt256 due_fees_collected;
  block::CurrencyCollection credit;
};

struct ComputePhase {
  enum { sk_none, sk_no_state, sk_bad_state, sk_no_gas, sk_suspended };
  int skip_reason{sk_none};
  bool success{false};
  bool msg_state_used{false};
  bool account_activated{false};
  bool out_of_gas{false};
  bool accepted{false};
  td::RefInt256 gas_fees;
  td::uint64 gas_used, gas_max, gas_limit, gas_credit;
  int mode;
  int exit_code;
  int exit_arg;
  int vm_steps;
  ton::Bits256 vm_init_state_hash, vm_final_state_hash;
  Ref<vm::Cell> in_msg;
  Ref<vm::Cell> new_data;
  Ref<vm::Cell> actions;
  std::string vm_log;
};

struct ActionPhase {
  bool success{false};
  bool valid{false};
  bool no_funds{false};
  bool code_changed{false};
  bool action_list_invalid{false};
  bool acc_delete_req{false};
  bool state_exceeds_limits{false};
  enum { acst_unchanged = 0, acst_frozen = 2, acst_deleted = 3 };
  int acc_status_change{acst_unchanged};
  td::RefInt256 total_fwd_fees;     // all fees debited from the account
  td::RefInt256 total_action_fees;  // fees credited to validators in this action phase
  int result_code;
  int result_arg;
  int tot_actions;
  int spec_actions;
  int skipped_actions;
  int msgs_created;
  Ref<vm::Cell> new_code;
  td::BitArray<256> action_list_hash;
  block::CurrencyCollection remaining_balance, reserved_balance;
  std::vector<Ref<vm::Cell>> action_list;  // processed in reverse order
  std::vector<Ref<vm::Cell>> out_msgs;
  ton::LogicalTime end_lt;
  unsigned long long tot_msg_bits{0}, tot_msg_cells{0};
};

struct BouncePhase {
  bool ok{false};
  bool nofunds{false};
  unsigned long long msg_bits, msg_cells;
  unsigned long long fwd_fees, fwd_fees_collected;
  Ref<vm::Cell> out_msg;
};

struct Account {
  enum { acc_nonexist = 0, acc_uninit = 1, acc_frozen = 2, acc_active = 3, acc_deleted = 4 };
  int status{acc_nonexist}, orig_status{acc_nonexist};
  bool is_special{false};
  bool tick{false};
  bool tock{false};
  bool split_depth_set_{false};
  unsigned char split_depth_{0};
  int verbosity{3 * 0};
  ton::UnixTime now_{0};
  ton::WorkchainId workchain{ton::workchainInvalid};
  td::BitArray<32> addr_rewrite;     // rewrite (anycast) data, split_depth bits
  ton::StdSmcAddress addr;           // rewritten address (by replacing a prefix of `addr_orig` with `addr_rewrite`); it is the key in ShardAccounts
  ton::StdSmcAddress addr_orig;      // address indicated in smart-contract data (must coincide with hash of StateInit)
  Ref<vm::CellSlice> my_addr;        // address as stored in the smart contract (MsgAddressInt); corresponds to `addr_orig` + anycast info
  Ref<vm::CellSlice> my_addr_exact;  // exact address without anycast info; corresponds to `addr` and has no anycast (rewrite) info
  ton::LogicalTime last_trans_end_lt_;
  ton::LogicalTime last_trans_lt_;
  ton::Bits256 last_trans_hash_;
  ton::LogicalTime block_lt;
  ton::UnixTime last_paid;
  vm::CellStorageStat storage_stat;
  block::CurrencyCollection balance;
  td::RefInt256 due_payment;
  Ref<vm::Cell> orig_total_state;  // ^Account
  Ref<vm::Cell> total_state;       // ^Account
  Ref<vm::CellSlice> storage;      // AccountStorage
  Ref<vm::CellSlice> inner_state;  // StateInit
  ton::Bits256 state_hash;         // hash of StateInit for frozen accounts
  Ref<vm::Cell> code, data, library, orig_library;
  std::vector<LtCellRef> transactions;
  Account() = default;
  Account(ton::WorkchainId wc, td::ConstBitPtr _addr) : workchain(wc), addr(_addr) {
  }
  Account(ton::WorkchainId wc, td::ConstBitPtr _addr, int depth)
      : split_depth_set_(true), split_depth_((unsigned char)depth), workchain(wc), addr(_addr) {
  }
  block::CurrencyCollection get_balance() const {
    return balance;
  }
  bool set_address(ton::WorkchainId wc, td::ConstBitPtr new_addr);
  bool unpack(Ref<vm::CellSlice> account, Ref<vm::CellSlice> extra, ton::UnixTime now, bool special = false);
  bool init_new(ton::UnixTime now);
  bool deactivate();
  bool recompute_tmp_addr(Ref<vm::CellSlice>& tmp_addr, int split_depth, td::ConstBitPtr orig_addr_rewrite) const;
  td::RefInt256 compute_storage_fees(ton::UnixTime now, const std::vector<block::StoragePrices>& pricing) const;
  bool is_masterchain() const {
    return workchain == ton::masterchainId;
  }
  bool belongs_to_shard(ton::ShardIdFull shard) const;
  bool store_acc_status(vm::CellBuilder& cb, int status) const;
  bool store_acc_status(vm::CellBuilder& cb) const {
    return store_acc_status(cb, status);
  }
  void push_transaction(Ref<vm::Cell> trans_root, ton::LogicalTime trans_lt);
  bool libraries_changed() const;
  bool create_account_block(vm::CellBuilder& cb);  // stores an AccountBlock with all transactions

 protected:
  friend struct transaction::Transaction;
  bool set_split_depth(int split_depth);
  bool check_split_depth(int split_depth) const;
  bool forget_split_depth();
  bool init_rewrite_addr(int split_depth, td::ConstBitPtr orig_addr_rewrite);

 private:
  bool unpack_address(vm::CellSlice& addr_cs);
  bool unpack_storage_info(vm::CellSlice& cs);
  bool unpack_state(vm::CellSlice& cs);
  bool parse_maybe_anycast(vm::CellSlice& cs);
  bool store_maybe_anycast(vm::CellBuilder& cb) const;
  bool compute_my_addr(bool force = false);
};

namespace transaction {
struct Transaction {
  static constexpr unsigned max_allowed_merkle_depth = 2;
  enum {
    tr_none,
    tr_ord,
    tr_storage,
    tr_tick,
    tr_tock,
    tr_split_prepare,
    tr_split_install,
    tr_merge_prepare,
    tr_merge_install
  };
  int trans_type{tr_none};
  bool was_deleted{false};
  bool was_frozen{false};
  bool was_activated{false};
  bool was_created{false};
  bool bounce_enabled{false};
  bool in_msg_extern{false};
  bool use_msg_state{false};
  bool is_first{false};
  bool orig_addr_rewrite_set{false};
  bool new_tick;
  bool new_tock;
  signed char new_split_depth{-1};
  ton::UnixTime now;
  int acc_status;
  int verbosity{3 * 0};
  int in_msg_type{0};
  const Account& account;                     // only `commit` method modifies the account
  Ref<vm::CellSlice> my_addr, my_addr_exact;  // almost the same as in account.*
  ton::LogicalTime start_lt, end_lt;
  block::CurrencyCollection balance, original_balance;
  block::CurrencyCollection msg_balance_remaining;
  td::RefInt256 due_payment;
  td::RefInt256 in_fwd_fee, msg_fwd_fees;
  block::CurrencyCollection total_fees{0};
  block::CurrencyCollection blackhole_burned{0};
  ton::UnixTime last_paid;
  Ref<vm::Cell> root;
  Ref<vm::Cell> new_total_state;
  Ref<vm::CellSlice> new_storage;
  Ref<vm::CellSlice> new_inner_state;
  Ref<vm::Cell> new_code, new_data, new_library;
  Ref<vm::Cell> in_msg, in_msg_state;
  Ref<vm::CellSlice> in_msg_body;
  Ref<vm::Cell> in_msg_library;
  td::BitArray<256> frozen_hash;
  td::BitArray<32> orig_addr_rewrite;
  std::vector<Ref<vm::Cell>> out_msgs;
  std::unique_ptr<StoragePhase> storage_phase;
  std::unique_ptr<CreditPhase> credit_phase;
  std::unique_ptr<ComputePhase> compute_phase;
  std::unique_ptr<ActionPhase> action_phase;
  std::unique_ptr<BouncePhase> bounce_phase;
  vm::CellStorageStat new_storage_stat;
  Transaction(const Account& _account, int ttype, ton::LogicalTime req_start_lt, ton::UnixTime _now,
              Ref<vm::Cell> _inmsg = {});
  bool unpack_input_msg(bool ihr_delivered, const ActionPhaseConfig* cfg);
  bool check_in_msg_state_hash();
  bool prepare_storage_phase(const StoragePhaseConfig& cfg, bool force_collect = true, bool adjust_msg_value = false);
  bool prepare_credit_phase();
  bool compute_gas_limits(ComputePhase& cp, const ComputePhaseConfig& cfg);
  Ref<vm::Stack> prepare_vm_stack(ComputePhase& cp);
  std::vector<Ref<vm::Cell>> compute_vm_libraries(const ComputePhaseConfig& cfg);
  bool prepare_compute_phase(const ComputePhaseConfig& cfg);
  bool prepare_action_phase(const ActionPhaseConfig& cfg);
  td::Status check_state_limits(const ActionPhaseConfig& cfg);
  bool prepare_bounce_phase(const ActionPhaseConfig& cfg);
  bool compute_state();
  bool serialize();
  td::uint64 gas_used() const {
    return compute_phase ? compute_phase->gas_used : 0;
  }

  td::Result<vm::NewCellStorageStat::Stat> estimate_block_storage_profile_incr(
      const vm::NewCellStorageStat& store_stat, const vm::CellUsageTree* usage_tree) const;
  bool update_block_storage_profile(vm::NewCellStorageStat& store_stat, const vm::CellUsageTree* usage_tree) const;
  bool would_fit(unsigned cls, const block::BlockLimitStatus& blk_lim_st) const;
  bool update_limits(block::BlockLimitStatus& blk_lim_st, bool with_size = true) const;

  Ref<vm::Cell> commit(Account& _account);  // _account should point to the same account
  LtCellRef extract_out_msg(unsigned i);
  NewOutMsg extract_out_msg_ext(unsigned i);
  void extract_out_msgs(std::vector<LtCellRef>& list);

 private:
  Ref<vm::Tuple> prepare_vm_c7(const ComputePhaseConfig& cfg) const;
  bool prepare_rand_seed(td::BitArray<256>& rand_seed, const ComputePhaseConfig& cfg) const;
  int try_action_set_code(vm::CellSlice& cs, ActionPhase& ap, const ActionPhaseConfig& cfg);
  int try_action_change_library(vm::CellSlice& cs, ActionPhase& ap, const ActionPhaseConfig& cfg);
  int try_action_send_msg(const vm::CellSlice& cs, ActionPhase& ap, const ActionPhaseConfig& cfg, int redoing = 0);
  int try_action_reserve_currency(vm::CellSlice& cs, ActionPhase& ap, const ActionPhaseConfig& cfg);
  bool check_replace_src_addr(Ref<vm::CellSlice>& src_addr) const;
  bool check_rewrite_dest_addr(Ref<vm::CellSlice>& dest_addr, const ActionPhaseConfig& cfg,
                               bool* is_mc = nullptr) const;
  bool serialize_storage_phase(vm::CellBuilder& cb);
  bool serialize_credit_phase(vm::CellBuilder& cb);
  bool serialize_compute_phase(vm::CellBuilder& cb);
  bool serialize_action_phase(vm::CellBuilder& cb);
  bool serialize_bounce_phase(vm::CellBuilder& cb);
  bool unpack_msg_state(bool lib_only = false);
};
}  // namespace transaction

struct FetchConfigParams {
static td::Status fetch_config_params(const block::Config& config,
                                      Ref<vm::Cell>* old_mparams,
                                      std::vector<block::StoragePrices>* storage_prices,
                                      StoragePhaseConfig* storage_phase_cfg,
                                      td::BitArray<256>* rand_seed,
                                      ComputePhaseConfig* compute_phase_cfg,
                                      ActionPhaseConfig* action_phase_cfg,
                                      td::RefInt256* masterchain_create_fee,
                                      td::RefInt256* basechain_create_fee,
                                      ton::WorkchainId wc,
                                      ton::UnixTime now);
};

}  // namespace block


  ```
</details>

This code appears to be part of the TON (The Open Network) blockchain implementation, specifically in the `block` namespace. Let's break down the key components and provide a summary:

1. **Header Inclusions:**
   - Includes various headers such as `refcnt.hpp`, `refint.h`, `cells.h`, `cellslice.h`, `dict.h`, `boc.h`, and others, which are likely part of the TON blockchain codebase.

2. **Namespace:**
   - The code is enclosed within the `block` namespace.

3. **Structs and Types:**
   - Defines several structs like `CollatorError`, `LtCellRef`, `LtCellCompare`, `NewOutMsg`, `StoragePhaseConfig`, `StoragePhase`, `ComputePhaseConfig`, `ComputePhase`, `ActionPhaseConfig`, `ActionPhase`, `BouncePhase`, and `Account`.
   - These structs seem to represent various aspects of the blockchain, including storage, computation, actions, and account details.

4. **Functions and Methods:**
   - Contains various member functions within the `Account` struct, such as `set_address`, `unpack`, `init_new`, `deactivate`, and others. These methods likely handle account-related operations.

5. **Transaction Struct:**
   - Defines a `Transaction` struct in the `transaction` namespace, with methods related to processing transactions. It includes logic for storage, credit, compute, action, and bounce phases.

6. **FetchConfigParams Struct:**
   - Defines a `FetchConfigParams` struct with a static method to fetch configuration parameters from the blockchain.

7. **Namespace Aliases:**
   - Uses aliases for types from other namespaces, such as `Ref` and `td` (possibly related to Telegram Database).

8. **Constants and Enumerations:**
   - Contains constants like `gas_infty` and enumerations for transaction types.

9. **Helper Functions:**
   - Includes functions for various tasks, such as parsing gas limits and prices, looking up libraries, and checking if an address is suspended.

Overall, this code seems to be a crucial part of the TON blockchain implementation, dealing with account management, transactions, and various phases involved in processing transactions on the blockchain. If you have specific questions or if there's a particular part you'd like more detail on, feel free to ask!