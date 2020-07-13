\textbf{Methodology for KPI calculation}
[\textbf{Dominique}: Mina, can you please update this document with the latest changes you made to the sampling and collection of metrics? Thanks]

Measurement reporting is done on two levels:
\begin{itemize}
\item  UDP application level on the mote: UDP packet rate is a 50\% probability of transmission, once per minute (effectively, once per 2 minutes on average). It carries the folloing info:
\begin{itemize}
\item Radio On duty cycle
\item Radio Tx duty cycle
\item src_id
\item counter
\item ASN
\item buffersize 
\begin{itemize}
\item \textbf{Comment}: Dominique suggests keeping an internal Max packets-in-buffer length to the mote for accurate sampling.
\end{itemize}
\item numCellsUsedTx
\item numCellsUsedRx
\item numNeighbors
\item dagRank
\end{itemize}
\item \textbf{At OpenVisualizer level:}
\begin{itemize}
\item The OV keeps track of all UDP packets coming from each mote in addition to all the serial output coming from the motes. Network level analytics are conducted at that level such as all the averages and the RPL information. }
\end{itemize}
\end{itemize}

\textbf{Offline Processing} [UPDATED]
\begin{itemize}
\item Sliding window processing: UDP packets are parsed, sorted, duplicates removed, and filtered by a sliding time window. The window width is 3 minutes (configurable) and it is sliding at 1 second per data point (configurable). For each window, there are two modes of plotting:
\begin{itemize}
\item Capturing average, median, mean, ,min, max, stdev, first quartile, and third quartile for all data points of all the mote data in this window for time-series line plots 
\item Capturing the raw sequence for a CDF plot
\end{itemize}
\end{itemize}

-\textbf{ Average Duty cycle: } [UPDATED]
    -\textbf{ DC calculation}: for each mote, and the end of each slot, if the radio was on (whether for Tx or Rx), the number of tics during which the radio was on is captured and appended to the statistics ieee154e_stats.numTicsOn. The total duration of the slot is also appended to ieee154e_stats.numTicsTotal. When a UDP packet is sent, those counters are reset. 
\begin{itemize}
\item \textbf{Comment}:  Dominique suggests using a sliding window for the averaging, instead of averaging for all the records of each mote. 
\item \textbf{Comment}: Quentin suggests relying more on CDF plots for parameter reporting instead of averages/medians to avoid the assumption that the distribution is uniform. I already have logged the individual uinject packets. With more work, I can obtain those plots if you agree to. [\textbf{Dominique}: I agree, thanks]
\item \textbf{Mina} Done!
\end{itemize}

- \textbf{Average Tx Duty Cycle} : The same as DC calculation, except that at each slot, the tics are calculated between the beginning and the end of each transmission (whether data, ack, or EB).
- \textbf{Average Packet Buffer Occupancy:}
    - A function is created under the openqueue component which counts openqueue entries whose owner is not NULL. The function is called every time a uinject packet is being composed. 

- \textbf{Average DAG Rank}: mote's DAGRank is retrieved via the uinject packet from the icmpv6rpl component. (Dominique notes for future to capture this from the serial port data)
\begin{itemize}
\item \textbf{Comment}:  Dominique suggests capturing this from serial port for more accurate sampling
\end{itemize}

- \textbf{Average Latency}: When uinject packet is composed, it retrieves the current Absolute Slot Number (ASN) from the ieee154e component. When the packet is received by the root, it gets the current ASN of the root and calculate the ASN difference. This difference is multiplied by the slot duration to get the latency in seconds. Average calculated as usual.

- \textbf{Average Number of Neighbors}: when uinject packet is being composed, it calls a function that looks at the neighbor table and counts all neighbors where the "used" flag is set to true. Average calculated as usual.
\begin{itemize}
\item \textbf{Comment}:  Dominique suggests capturing this from serial port for more accurate sampling
\end{itemize}
- \textbf{Average End-to-End PDR}: At OV the side, each uinject packet includes a counter. The root may receive a pcaket more than once. It keeps a record of all the packets, then it sorts the records for each mote. The PDR is calculated for each mote as : length of the set of counters of total packets received/(latest counter- first counter).  
\begin{itemize}
\item \textbf{Comment}:  Dominique suggests capturing this from serial port for more accurate sampling
\end{itemize}
- \textbf{Network Size}: OV keeps track of the RPL DAO messages. Each message contains  parent/child pair. These messages are used to construct a global Tree on the OV side. If an update is not heard from the node after 900 seconds since the last update, it is assumed to be disconnected from the RPL DAG and is removed from the graph.

-\textbf{ Network Churn}: At OV side, a function is called periodically which retrieves the latest RPL graph. It compares it against the existing graph and computes the churn as: # of new motes + # of removed nodes + # of motes that changed their parents in the new graph. 

- \textbf{Time to first packet CDF}: OV captures the time t1 when the root is set. Then when packets start arriving, it checks if it is the first arrival from that mote src_id. If so, it captures the time of arrival t2 and stores the duration t1-t2. 
\begin{itemize}
\item \textbf{Caution}: this includes the time for UDP packet transfer over MQTT to the OV client. The set root event is captured at OV so the comparison is done with this timestamp at OV side. 
\end{itemize}


\textit{Regarding a couple of questions that Dominique and Quentin raised on whether the measurement mechanism can impact the network performance:}

- The uinject (UDP inject) packet is being generated anyway since the measurements are about the network performance with generated packets. Instead of using a payload of random bytes, we just use values from the mote itself. All those measurements can be obtained from the serial port, but the uinject packet is being generated anyway so there is no point of redoing that process on the serial port. [\textbf{Dominique}: this is probably true as far as UDP packet size is concerned (btw, what is the UDP payload size when it is not used to transport metrics?). However, the fact that metrics are sampled at the moment a UDP packet is created is \textbf{very different }from the CPU sending proactively over the serial port any change in the metrics of interest].The only parameter that is time-sensitive is the ASN of the packet transmission, which is capture during packet composition anyway.

-  The mote writes different stats on the serial port slightly before the end of each slot until the beginning of the new slot. During slot execution, serial communications are inhibited. So serial communications are well separated from performance. 
[\textbf{Dominique}: it would be useful to know what it is the time allowed for serial communication, what is the serial port data rate. Hence, what amount of data can be sent at every slot? So that we can use it to the fullest, but not exceed its capacity.]
