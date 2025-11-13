#!/usr/bin/env python3

import sys
import time
import math
from easysnmp import Session
from easysnmp.exceptions import EasySNMPTimeoutError, EasySNMPNoSuchInstanceError, EasySNMPError, EasySNMPConnectionError

def main():
    global last_timestamp, current_value, no_of_samples

    # Command-line arguments
    agent_info = sys.argv[1]
    sampling_frequency = float(sys.argv[2])
    no_of_samples = int(sys.argv[3])
    ip, port, community = agent_info.split(':')

    # Calculating the sampling interval from the sampling frequency
    sampling_interval = 1 / sampling_frequency

    # Initializing OID list
    oid_list = ['1.3.6.1.2.1.1.3.0']
    for i in range(4, len(sys.argv)):
        oid_list.append(sys.argv[i])

    last_timestamp = 0
    current_value = []

    # Initializing SNMP session
    try:
        prob_session = Session(hostname=ip, remote_port=int(port), community=community, version=2, timeout=1, retries=3)
    except EasySNMPConnectionError as e:
        print(f"ERROR - Failed to establish SNMP connection: {e}")
        sys.exit(1)

    # Determine sampling mode
    if no_of_samples == -1:
        run_continuous_monitoring(prob_session, oid_list, sampling_interval)
    else:
        run_sample_collection(prob_session, no_of_samples, oid_list, sampling_interval)

# Fetching SNMP data
def fetch_data(prob_session, oid_list):
    try:
        return prob_session.get(oid_list)
    except EasySNMPTimeoutError:
        print("Connection timed out. Retrying in 5 seconds...")
        time.sleep(5)
        return None
    except EasySNMPNoSuchInstanceError as e:
        print(f"ERROR - OID does not exist: {e}")
        return None
    except EasySNMPError as e:
        print(f"ERROR - General SNMP error: {e}")
        return None

# Process fetched data and calculate rates
def process_data(response):
    global current_value, last_timestamp

    next_values = []
    rate_values = []
    current_timestamp = time.time()

    if last_timestamp == 0:
        last_timestamp = current_timestamp

    for index in range(1, len(response)):
        if response[index].value != 'NOSUCHINSTANCE' and response[index].value != 'NOSUCHOBJECT':    
            next_values.append(int(response[index].value))
            if len(current_value) > 0:
                previous_value = int(current_value[index - 1])
                next_value = int(next_values[index - 1])
                numerator = next_value - previous_value
                denominator = round(current_timestamp - last_timestamp, 1)

                if denominator == 0:
                    rate = 0
                else:
                    rate = math.ceil(numerator / denominator)

                if rate < 0:  # Handle counter wrap-around
                    if response[index].snmp_type == 'COUNTER64':
                        numerator += 2 ** 64
                    elif response[index].snmp_type in ['COUNTER32', 'COUNTER']:
                        numerator += 2 ** 32
                    rate = math.ceil(numerator / denominator)

                rate_values.append(rate)

    current_value = next_values
    last_timestamp = current_timestamp

    return rate_values, current_timestamp

# Print object ID value and rate
def print_object_id_value(timestamp, rates):
    if rates and not all(rate == 0.0 for rate in rates):
        print(f"{int(timestamp)}", end=" | ")
        print(" | ".join(map(str, rates)))

# Run continuous monitoring
def run_continuous_monitoring(prob_session, oid_list, sampling_interval):
    last_values = []
    while True:
        start_time = time.time()
        response = fetch_data(prob_session, oid_list)
        if response is not None:
            last_values, last_timestamp = process_data(response)
            print_object_id_value(last_timestamp, last_values)
        end_time = time.time()
        
        # Sleep to maintain the desired sampling interval
        elapsed_time = end_time - start_time
        if sampling_interval > elapsed_time:
            time.sleep(sampling_interval - elapsed_time)
        else:
            x = math.ceil(elapsed_time/sampling_interval)
            time.sleep((x*sampling_interval)-elapsed_time)  

# Run specific no of samples
def run_sample_collection(prob_session, sample_count, oid_list, sampling_interval):
    last_values = []
    for j in range(sample_count + 1):  # Increase the sample count by 1
        start_time = time.time()
        response = fetch_data(prob_session, oid_list)
        if response is not None:
            last_values, last_timestamp = process_data(response)
            print_object_id_value(last_timestamp, last_values)
        end_time = time.time()
        
        # Sleep to maintain the desired sampling interval
        elapsed_time = end_time - start_time
        if sampling_interval > elapsed_time:
            time.sleep(sampling_interval - elapsed_time)
        else:
            x = math.ceil(elapsed_time/sampling_interval)
            time.sleep((x*sampling_interval)-elapsed_time)  

if __name__ == "__main__":
    main()
