# 1.NACK的含义

丢包重传(NACK)是抵抗网络错误的重要手段。NACK在接收端检测到数据丢包后，发送NACK报文到发送端；发送端根据NACK报文中的序列号，在发送缓冲区找到对应的数据包，重新发送到接收端。NACK需要发送端，发送缓冲区的支持。

WebRTC中支持音频和视频的NACK重传。我们这里只分析nack机制，不分析jitterbuffer或者neteq的更多实现。

# 2.WebRTC中NACK请求发送的条件

这里以视频为例。

下面是webrtc中接收端触发nack的条件，我们看下nack_module.cc文件中OnReceivedPacket的实现。

```c++
void NackModule::OnReceivedPacket(const VCMPacket& packet) {

 rtc::CritScope lock(&crit_);

 if (!running_)

  return;

 //获取包的seqnum

 uint16_t seq_num = packet.seqNum;

 // TODO(philipel): When the packet includes information whether it is

 //         retransmitted or not, use that value instead. For

 //         now set it to true, which will cause the reordering

 //         statistics to never be updated.

 bool is_retransmitted = true;

 //判断第一帧是不是关键帧

 bool is_keyframe = packet.isFirstPacket && packet.frameType == kVideoFrameKey;

//拿到第一个包的时候判断，把第一个包的seqnum赋值给最新的last_seq_num,如果是关键帧的话，插入到关键帧列表中，同时把initialized_设置为true

 if (!initialized_) {

  last_seq_num_ = seq_num;

  if (is_keyframe)

   keyframe_list_.insert(seq_num);

  initialized_ = true;

  return;

 }

 if (seq_num == last_seq_num_)

  return;

//判断有无乱序，乱序了，如来1,2,3,6包，然后来4包，就乱序了，就把4从nack_list中去掉，不再通知发送端重新发送4了

 if (AheadOf(last_seq_num_, seq_num)) {

  // An out of order packet has been received.

  //把重新收到的包从nack_list中移除掉

  nack_list_.erase(seq_num);

  if (!is_retransmitted)

   UpdateReorderingStatistics(seq_num);

  return;

 } else {

 //没有乱序，如1,2,3,6包，就把（3+1,6）之间的包加入到nack_list中

  AddPacketsToNack(last_seq_num_ + 1, seq_num);

  last_seq_num_ = seq_num;

  // Keep track of new keyframes.

  if (is_keyframe)

   keyframe_list_.insert(seq_num);

  // And remove old ones so we don't accumulate keyframes.

  auto it = keyframe_list_.lower_bound(seq_num - kMaxPacketAge);

  if (it != keyframe_list_.begin())

   keyframe_list_.erase(keyframe_list_.begin(), it);

  // Are there any nacks that are waiting for this seq_num.

  //从nack_list 中取出需要发送 NACK 的序号列表, 如果某个 seq 请求次数超过 kMaxNackRetries = 10次则会从nack_list 中删除.

  std::vector<uint16_t> nack_batch = GetNackBatch(kSeqNumOnly);

  //LOG(LS_INFO) << "nack_batch size[" << nack_batch.size() << "].";

  //在 NackModule 中触发使用 NackSender::SednNack 发送 NACK 请求

  if (!nack_batch.empty())

   nack_sender_->SendNack(nack_batch);

 }

}
```



我们继续跟踪流程看下AddPacketsToNack函数的实现

```c++



void NackModule::AddPacketsToNack(uint16_t seq_num_start,

​                 uint16_t seq_num_end) {

 //LOG(LS_INFO) << "AddPacketsToNack. "

 //       << "start seq[" << seq_num_start

 //       << "],end seq[" << nack_list_.lower_bound(seq_num_end - kMaxPacketAge);

 nack_list_.erase(nack_list_.begin(), it);

 // If the nack list is too large, remove packets from the nack list until

 // the latest first packet of a keyframe. If the list is still too large,

 // clear it and request a keyframe.

 uint16_t num_new_nacks = ForwardDiff(seq_num_start, seq_num_end);

 if (nack_list_.size() + num_new_nacks > kMaxNackPackets) {

  while (RemovePacketsUntilKeyFrame() &&

​      nack_list_.size() + num_new_nacks > kMaxNackPackets) {

  }

  if (nack_list_.size() + num_new_nacks > kMaxNackPackets) {

   nack_list_.clear();

   LOG(LS_WARNING) << "NACK list full, clearing NACK"

​             " list and requesting keyframe.";

  //触发关键帧请求

   keyframe_request_sender_->RequestKeyFrameNack();

   return;

  }

 }

 for (uint16_t seq_num = seq_num_start; seq_num != seq_num_end; ++seq_num) {

  NackInfo nack_info(seq_num, seq_num + WaitNumberOfPackets(0.5),

​            clock_->TimeInMilliseconds());

  RTC_DCHECK(nack_list_.find(seq_num) == nack_list_.end());

  nack_list_[seq_num] = nack_info;

 }

 //LOG(LS_INFO) << "nack_list size[" << nack_list_.size() << "]";

}

```

我们可以看到AddPacketsToNack()函数主要实现了：

nack_list 的最大容量为 kMaxNackPackets = 1000, 如果满了会删除最后一个 KeyFrame 之前的所有nacked 序号, 如果删除之后还是满的那么清空 nack_list 并请求KeyFrame。

我们继续跟踪流程，我们看下GetNackBatch函数实现

```c++

std::vector<uint16_t> NackModule::GetNackBatch(NackFilterOptions options) {

 bool consider_seq_num = options != kTimeOnly;

 bool consider_timestamp = options != kSeqNumOnly;

 int64_t now_ms = clock_->TimeInMilliseconds();

 std::vector<uint16_t> nack_batch;

 auto it = nack_list_.begin();

 //LOG(LS_INFO) << "nack_list size[" << nack_list_.size() << "]";

 while (it != nack_list_.end()) {

  bool delay_timed_out =

​    now_ms - it->second.created_at_time >= kDefaultSendNackDelayMs;

​    //只考虑时间模式

​    //当前序号是第一次发送(本地记录的send_at_time == -1)

​    //当前最新收到的包序号在这个需要发送NAKC的序号的后面(避免当前还在收之前没收到的包)

//比如当前最新收到100, 当前检测是否需要发送NACK的序号为小于等于100的才满足条件, 比如 99

  if (delay_timed_out && consider_seq_num && it->second.sent_at_time == -1 &&

​    AheadOrAt(last_seq_num_, it->second.send_at_seq_num)) {

   nack_batch.emplace_back(it->second.seq_num);

   ++it->second.retries;

   it->second.sent_at_time = now_ms;

   //从nack_list 中取出需要发送 NACK 的序号列表, 如果某个 seq 请求次数超过 kMaxNackRetries = 10次则会从nack_list 中删除

   if (it->second.retries >= kMaxNackRetries) {

​    LOG(LS_WARNING) << "Sequence number " << it->second.seq_num

​            << " removed from NACK list due to max retries.";

​    //从nack_list_列表中移除

​    it = nack_list_.erase(it);

   } else {

​    ++it;

   }

   continue;

  }

  //只考虑时间模式

  //发送nack的条件变成，该序号上次发送NACK的时间到当前时间要超过1个RTT(该序号一次也没发送过NACK(send_at_time == -1)也满足

  if (delay_timed_out && consider_timestamp && it->second.sent_at_time + rtt_ms_ <= now_ms) {

   nack_batch.emplace_back(it->second.seq_num);

   ++it->second.retries;

   it->second.sent_at_time = now_ms;

   if (it->second.retries >= kMaxNackRetries) {

​    LOG(LS_WARNING) << "Sequence number " << it->second.seq_num

​            << " removed from NACK list due to max retries.";

​    it = nack_list_.erase(it);

   } else {

​    ++it;

   }

   continue;

  }

  ++it;

 }

 return nack_batch;

}

```

从上面GetNackBatch函数我们可以知道，获取nack_list存在2种控制逻辑。

# 3.WebRTC中处理NACK请求的实现

1. 首先是正常的 RTCP 处理流程: RTCPReceiver 中解析处理RTCP, 在 rtcp_receiver.cc中的TriggerCallbacksFromRtcpPacket函数中处理不同的RTCP消息.

2. 如果nackSequenceNumbers.size大于0，则触发 RtpRtcp 对象的ModuleRtpRtcpImpl::OnReceivedNack 处理流程。

我们看下OnReceivedNACK/rtp_rtcp_imp.cc函数

```c++

void ModuleRtpRtcpImpl::OnReceivedNACK(

  int64_t id, const std::list<uint16_t>& nack_sequence_numbers) {

  //将丢包的序号 记录到PacketLossStats, 获取RTT后进入 RTPSedner.OnReceivedNack.

 for (uint16_t nack_sequence_number : nack_sequence_numbers) {

  send_loss_stats_.AddLostPacket(nack_sequence_number);

 }

 if (!rtp_sender_.StorePackets() ||

   nack_sequence_numbers.size() == 0) {

  return;

 }

 // Use RTT from RtcpRttStats class if provided.

 int64_t rtt = rtt_ms();

 if (rtt == 0) {

  rtcp_receiver_.RTT(rtcp_receiver_.RemoteSSRC(), NULL, &rtt, NULL, NULL);

 }

 rtp_sender_.OnReceivedNACK(id, nack_sequence_numbers, rtt);

}

```

我们继续跟踪流程，看下rtp_sender.cc下OnReceivedNACK()函数

```c++

void RTPSender::OnReceivedNACK(int64_t id,

​                const std::list<uint16_t>& nack_sequence_numbers,

​                int64_t avg_rtt) {

 TRACE_EVENT2(TRACE_DISABLED_BY_DEFAULT("webrtc_rtp"),

​        "RTPSender::OnReceivedNACK", "num_seqnum",

​        nack_sequence_numbers.size(), "avg_rtt", avg_rtt);

 const int64_t now = clock_->TimeInMilliseconds();

 uint32_t bytes_re_sent = 0;

 uint32_t target_bitrate = GetTargetBitrate();

  //比特率限制检查

 // Enough bandwidth to send NACK?

 if (!ProcessNACKBitRate(now)) {

  LOG(LS_INFO) << "NACK bitrate reached. Skip sending NACK response. Target "

​         << target_bitrate;

  return;

 }

 for (std::list<uint16_t>::const_iterator it = nack_sequence_numbers.begin();

   it != nack_sequence_numbers.end(); ++it) {

  const int32_t bytes_sent = ReSendPacket(id, *it, 5 + avg_rtt);

  if (bytes_sent > 0) {

   bytes_re_sent += bytes_sent;

  } else if (bytes_sent == 0) {

   // The packet has previously been resent.

   // Try resending next packet in the list.

   continue;

  } else {

   // Failed to send one Sequence number. Give up the rest in this nack.

   LOG(LS_WARNING) << "Failed resending RTP packet " << *it

​           << ", Discard rest of packets";

   break;

  }

  // Delay bandwidth estimate (RTT * BW).

  if (target_bitrate != 0 && avg_rtt) {

   // kbits/s * ms = bits => bits/8 = bytes

   size_t target_bytes =

​     (static_cast<size_t>(target_bitrate / 1000) * avg_rtt) >> 3;

   if (bytes_re_sent > target_bytes) {

​    break; // Ignore the rest of the packets in the list.

   }

  }

 }

 if (bytes_re_sent > 0) {

  UpdateNACKBitRate(bytes_re_sent, now);

 }

}

```

我们继续看下ReSendPacket()函数重新发送数据的实现

```c++

int32_t RTPSender::ReSendPacket(int64_t id, uint16_t packet_id, int64_t min_resend_time) {

 size_t length = IP_PACKET_SIZE;

 uint8_t data_buffer[IP_PACKET_SIZE];

 int64_t capture_time_ms;

//从缓存包中去获取数据包

 if (!packet_history_.GetPacketAndSetSendTime(packet_id, min_resend_time, true,

​                        data_buffer, &length,

​                        &capture_time_ms)) {

  // Packet not found.

  LOG(LS_INFO) << "ReSendPacket not found.seq[" << packet_id << "].";

  return 0;

 }

//如果开启平滑发送的话

 if (paced_sender_) {

  RtpUtility::RtpHeaderParser rtp_parser(data_buffer, length);

  RTPHeader header;

  if (!rtp_parser.Parse(&header)) {

   assert(false);

   return -1;

  }

  // Convert from TickTime to Clock since capture_time_ms is based on

  // TickTime.

  int64_t corrected_capture_tims_ms = capture_time_ms + clock_delta_ms_;

  paced_sender_->InsertPacket(

​    id, RtpPacketSender::kNormalPriority, header.ssrc, header.sequenceNumber,

​    corrected_capture_tims_ms, length - header.headerLength, true);

  return length;

 }

 int rtx = kRtxOff;

 {

  rtc::CritScope lock(&send_critsect_);

  rtx = rtx_;

 }

 //重新发送数据

 if (!PrepareAndSendPacket(id, data_buffer, length, capture_time_ms,

​              (rtx & kRtxRetransmitted) > 0, true)) {

  return -1;

 }

 return static_cast<int32_t>(length);

}

```

3.通过上面我们可以知道，RTPSender中完成在PacketHistory中查找需要发送的RTP seq, 并决定重发时间. 重发也需要经过重发的比特率限制的检查. RTPSedner 初始化话时可以配置是否使用(PacedSend, 均匀发送), 最后检查重发格式(RtxStatus() 可以获取是否使用 RTX 封装)后使用 RTPSedner::PrepareAndSendPacket进行立即重发. 如果是使用 PacedSend, 则使用 PacedSender::InsertPacket 先加入发送列表中, 它的process会定时处理发送任务.

