const { spawn } = require('child_process'); // Use spawn for safer execution
const zabbixSender = require('zabbix-sender');

function useRegex(input) {
  let regex = /^([a-z0-9]+(-[a-z0-9]+)+)$/i;
  return regex.test(input);
}

// Configuration
const ZABBIX_SERVER = '10.20.169.30';
const ZABBIX_HOST = 'zabbix-proxy'; // This should match the hostname in Zabbix
const ZABBIX_KEY = 'zabbix';
const NAMESPACE = 'zabbix-testing';
const POD_NAME_PATTERN = 'pulse-4-gen-ai-inference-service-product-node-2';

try {
  // Get the pod name
  const process = spawn('kubectl', ['get', 'pods', '-n', NAMESPACE, '--no-headers', '-o', 'custom-columns=:metadata.name', '|', 'grep', POD_NAME_PATTERN]);

  let podName = '';
  process.stdout.on('data', (data) => {
    podName += data.toString().trim();
  });

  process.stderr.on('data', (data) => {
    console.error('Error getting pod name:', data.toString());
  });

  process.on('close', (code) => {
    if (code !== 0) {
      throw new Error('Failed to get pod name'); // Handle specific error
    } else if (!useRegex(podName)) {
      throw new Error('Pod name does not match the regex pattern');
    } else {
      // Execute df command
      const dfProcess = spawn('kubectl', ['exec', '-n', NAMESPACE, podName, '--', 'df', '-h', '/dev/shm']);

      let shmUtilizationOutput = '';
      dfProcess.stdout.on('data', (data) => {
        shmUtilizationOutput += data.toString();
      });

      dfProcess.stderr.on('data', (data) => {
        console.error('Error getting shm utilization:', data.toString());
      });

      dfProcess.on('close', (dfCode) => {
        if (dfCode !== 0) {
          throw new Error('Failed to get shm utilization'); // Handle specific error
        } else {
          // Extract shm utilization
          const shmUtilization = shmUtilizationOutput.split('\n')[1].split(/\s+/)[4].replace('%', '');

          // Send data to Zabbix (unchanged)
          const sender = new zabbixSender({ host: ZABBIX_SERVER });
          sender.addItem(ZABBIX_HOST, ZABBIX_KEY, shmUtilization);
          sender.send((err, res) => {
            if (err) {
              console.error('Error sending data to Zabbix:', err);
            } else {
              console.log('Data sent to Zabbix:', res);
            }
          });

          console.log(`Shared memory utilization for pod ${podName} is ${shmUtilization}%`);
        }
      });
    }
  });
} catch (error) {
  console.error('Error:', error);
}
