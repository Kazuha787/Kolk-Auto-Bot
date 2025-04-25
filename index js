require('dotenv').config();
const fs = require('fs');
const { ethers } = require('ethers');
const blessed = require('neo-blessed');
const chalk = require('chalk');
const { HttpsProxyAgent } = require('https-proxy-agent');

const COLORS = {
  GREEN: '#00ff00',
  YELLOW: '#ffff00',
  RED: '#ff0000',
  WHITE: '#ffffff',
  GRAY: '#808080',
  CYAN: '#00ffff',
  MAGENTA: '#ff00ff',
};

let proxies = [];
let privateKeys = [];

// Contract addresses and ABIs
const USDC_ADDRESS = '0xef84994ef411c4981328ffce5fda41cd3803fae4';
const R2USD_ADDRESS = '0x20c54c5f742f123abb49a982bfe0af47edb38756';
const SR2USD_ADDRESS = '0xbd6b25c4132f09369c354bee0f7be777d7d434fa';
const USDC_TO_R2USD_CONTRACT = '0x20c54c5f742f123abb49a982bfe0af47edb38756';
const R2USD_TO_USDC_CONTRACT = '0x07abd582df3d3472aa687a0489729f9f0424b1e3';
const STAKE_R2USD_CONTRACT = '0xbd6b25c4132f09369c354bee0f7be777d7d434fa';

const USDC_TO_R2USD_METHOD_ID = '0x095e7a95';
const R2USD_TO_USDC_METHOD_ID = '0x3df02124';
const STAKE_R2USD_METHOD_ID = '0x1a5f0f00';

const ERC20_ABI = [
  'function approve(address spender, uint256 amount) external returns (bool)',
  'function allowance(address owner, address spender) external view returns (uint256)',
  'function balanceOf(address account) external view returns (uint256)',
  'function decimals() external view returns (uint8)',
];

const NETWORKS = [
  {
    name: 'sepolia',
    chainId: 11155111,
    explorer: 'https://sepolia.etherscan.io',
    rpcUrls: [
      'https://ethereum-sepolia-rpc.publicnode.com',
      'https://rpc.sepolia.org',
    ],
  },
  {
    name: 'plume_testnet',
    chainId: 98867,
    explorer: 'https://testnet-explorer.plumenetwork.xyz',
    rpcUrls: [
      'https://testnet-rpc.plumenetwork.xyz',
      'https://testnet-explorer.plumenetwork.xyz/',
    ],
  },
  {
    name: 'arbitrum_sepolia',
    chainId: 421614,
    explorer: 'https://sepolia.arbiscan.io',
    rpcUrls: [
      'https://sepolia-rollup.arbitrum.io/rpc',
    ],
  },
];

// Create neo-blessed screen
const screen = blessed.screen({
  smartCSR: true,
  title: 'R2-AUTO-BOT',
  cursor: { color: COLORS.GREEN },
});

// Main container
const container = blessed.box({
  parent: screen,
  top: 0,
  left: 0,
  width: '100%',
  height: '100%',
  style: { bg: 'black', fg: COLORS.GREEN },
});

// Status bar
const statusBar = blessed.box({
  parent: container,
  top: 0,
  left: 0,
  width: '100%',
  height: 1,
  content: ' [R2-AUTO-BOT v2.0] - SYSTEM ONLINE ',
  style: { bg: COLORS.GREEN, fg: 'black', bold: true },
});

// Log window
const logWindow = blessed.log({
  parent: container,
  top: 1,
  left: 0,
  width: '70%',
  height: '90%',
  border: { type: 'line', fg: COLORS.GREEN },
  style: { fg: COLORS.GREEN, bg: 'black', scrollbar: { bg: COLORS.GREEN } },
  scrollable: true,
  scrollbar: true,
  tags: true,
  padding: { left: 1, right: 1 },
});

// Info panel
const infoPanel = blessed.box({
  parent: container,
  top: 1,
  right: 0,
  width: '30%',
  height: '90%',
  border: { type: 'line', fg: COLORS.GREEN },
  style: { fg: COLORS.GREEN, bg: 'black' },
  content: '{center}SYSTEM INFO{/center}\n\nInitializing...',
  tags: true,
});

// Input box
const inputBox = blessed.textbox({
  parent: container,
  bottom: 0,
  left: 0,
  width: '100%',
  height: 3,
  border: { type: 'line', fg: COLORS.GREEN },
  style: { fg: COLORS.GREEN, bg: 'black' },
  hidden: true,
  inputOnFocus: true,
});

// Key bindings
screen.key(['escape', 'q', 'C-c'], () => process.exit(0));

// Utility functions
function colorText(text, color) {
  return `{${color}-fg}${text}{/}`;
}

function isValidPrivateKey(key) {
  const cleanKey = key.startsWith('0x') ? key.slice(2) : key;
  return /^[0-9a-fA-F]{64}$/.test(cleanKey);
}

function loadProxies() {
  try {
    if (fs.existsSync('./proxies.txt')) {
      proxies = fs.readFileSync('./proxies.txt', 'utf8')
        .split('\n')
        .filter(line => line.trim().length > 0);
      logWindow.log(`${colorText(`Loaded ${proxies.length} proxies from proxies.txt`, COLORS.GREEN)}`);
    } else {
      logWindow.log(`${colorText('proxies.txt not found. Connecting directly.', COLORS.RED)}`);
    }
  } catch (error) {
    logWindow.log(`${colorText(`Failed to load proxies: ${error.message}`, COLORS.RED)}`);
  }
}

function loadPrivateKeys() {
  try {
    const envKeys = Object.keys(process.env).filter(key => key.startsWith('PRIVATE_KEY_'));
    if (envKeys.length > 0) {
      privateKeys = envKeys
        .map(key => process.env[key])
        .filter(key => key && key.trim().length > 0)
        .filter(key => {
          if (!isValidPrivateKey(key)) {
            logWindow.log(`${colorText(`Invalid private key format for ${key.slice(0, 6)}...: must be 64 hex characters`, COLORS.RED)}`);
            return false;
          }
          return true;
        });
    }
    if (privateKeys.length === 0) {
      logWindow.log(`${colorText('No valid private keys found in .env (PRIVATE_KEY_*)', COLORS.RED)}`);
      process.exit(1);
    }
    logWindow.log(`${colorText(`Loaded ${privateKeys.length} private key(s) from .env`, COLORS.GREEN)}`);
  } catch (error) {
    logWindow.log(`${colorText(`Failed to load private keys from .env: ${error.message}`, COLORS.RED)}`);
    process.exit(1);
  }
}

function getRandomProxy() {
  if (proxies.length === 0) return null;
  return proxies[Math.floor(Math.random() * proxies.length)];
}

function formatProxy(proxyString) {
  if (!proxyString) return null;
  let proxy = proxyString.trim();
  if (proxy.includes('://')) {
    proxy = proxy.split('://')[1];
  }
  let auth = '';
  let address = proxy;
  if (proxy.includes('@')) {
    const parts = proxy.split('@');
    auth = parts[0];
    address = parts[1];
  }
  const [host, port] = address.split(':');
  let username = '';
  let password = '';
  if (auth) {
    const authParts = auth.split(':');
    username = authParts[0];
    password = authParts.length > 1 ? authParts[1] : '';
  }
  return {
    host,
    port: parseInt(port, 10),
    auth: auth ? { username, password } : undefined,
  };
}

async function initializeWallet(privateKey, network) {
  try {
    const stopSpinner = showSpinner(
      `Connecting to ${network.name} network...`,
      `Connected to ${network.name} network!`,
      60
    );
    let provider;
    const proxyString = getRandomProxy();
    let lastError = null;
    for (const url of network.rpcUrls) {
      try {
        if (proxyString) {
          const proxyConfig = formatProxy(proxyString);
          logWindow.log(`${colorText(`Using proxy: ${proxyString} with RPC: ${url}`, COLORS.GRAY)}`);
          const agent = new HttpsProxyAgent({
            host: proxyConfig.host,
            port: proxyConfig.port,
            auth: proxyConfig.auth ? `${proxyConfig.auth.username}:${proxyConfig.auth.password}` : undefined,
          });
          provider = new ethers.providers.JsonRpcProvider(
            {
              url,
              agent,
            },
            { name: network.name, chainId: network.chainId }
          );
        } else {
          logWindow.log(`${colorText(`Using RPC: ${url}`, COLORS.GRAY)}`);
          provider = new ethers.providers.JsonRpcProvider(
            url,
            { name: network.name, chainId: network.chainId }
          );
        }
        await provider.getNetwork();
        logWindow.log(`${colorText(`Connected to RPC: ${url}`, COLORS.GREEN)}`);
        lastError = null;
        break;
      } catch (error) {
        lastError = error;
        logWindow.log(`${colorText(`Failed to connect to RPC ${url}: ${error.message}`, COLORS.RED)}`);
        continue;
      }
    }
    if (lastError) {
      throw new Error(`All RPCs failed for ${network.name}: ${lastError.message}`);
    }
    stopSpinner();
    const wallet = new ethers.Wallet(privateKey, provider);
    logWindow.log(`${colorText(`Wallet initialized on ${network.name}: ${wallet.address.slice(0, 6)}...${wallet.address.slice(-4)}`, COLORS.WHITE)}`);
    return { provider, wallet };
  } catch (error) {
    logWindow.log(`${colorText(`Failed to initialize wallet for key ${privateKey.slice(0, 6)}... on ${network.name}: ${error.message}`, COLORS.RED)}`);
    throw error;
  }
}

async function checkBalance(wallet, tokenAddress) {
  try {
    const tokenContract = new ethers.Contract(tokenAddress, ERC20_ABI, wallet);
    const balance = await tokenContract.balanceOf(wallet.address);
    const decimals = await tokenContract.decimals();
    return ethers.utils.formatUnits(balance, decimals);
  } catch (error) {
    logWindow.log(`${colorText(`Failed to check balance for token ${tokenAddress}: ${error.message}`, COLORS.RED)}`);
    return '0';
  }
}

async function checkEthBalance(wallet) {
  try {
    const balance = await wallet.provider.getBalance(wallet.address);
    return ethers.utils.formatEther(balance);
  } catch (error) {
    logWindow.log(`${colorText(`Failed to check ETH balance: ${error.message}`, COLORS.RED)}`);
    return '0';
  }
}

async function updateWalletInfo(wallets, network) {
  const walletInfo = [];
  for (const wallet of wallets) {
    const ethBalance = await checkEthBalance(wallet);
    const usdcBalance = await checkBalance(wallet, USDC_ADDRESS);
    const r2usdBalance = await checkBalance(wallet, R2USD_ADDRESS);
    const sr2usdBalance = network.name === 'sepolia' ? await checkBalance(wallet, SR2USD_ADDRESS) : 'N/A';
    walletInfo.push(
      `WALLET: ${wallet.address.slice(0, 6)}...${wallet.address.slice(-4)}\n` +
      `ETH: ${parseFloat(ethBalance).toFixed(4)}\n` +
      `USDC: ${parseFloat(usdcBalance).toFixed(2)}\n` +
      `R2USD: ${parseFloat(r2usdBalance).toFixed(2)}\n` +
      `sR2USD: ${network.name === 'sepolia' ? parseFloat(sr2usdBalance).toFixed(2) : 'N/A'}\n`
    );
  }
  infoPanel.setContent(
    '{center}{bold}SYSTEM INFO{/bold}{/center}\n\n' +
    walletInfo.join('---\n') +
    `{green-fg}STATUS: ONLINE{/green-fg}\n` +
    `{green-fg}NETWORK: ${network.name} (chainId: ${network.chainId}){/green-fg}`
  );
  screen.render();
}

async function estimateGas(wallet, tx) {
  try {
    const estimatedGas = await wallet.estimateGas(tx);
    // Add 20% buffer
    return estimatedGas.mul(120).div(100);
  } catch (error) {
    logWindow.log(`${colorText(`Failed to estimate gas: ${error.message}`, COLORS.RED)}`);
    return ethers.BigNumber.from('200000'); // Fallback gas limit
  }
}

async function approveToken(wallet, tokenAddress, spenderAddress, amount) {
  try {
    const tokenContract = new ethers.Contract(tokenAddress, ERC20_ABI, wallet);
    const decimals = await tokenContract.decimals();
    const currentAllowance = await tokenContract.allowance(wallet.address, spenderAddress);
    if (currentAllowance.gte(ethers.utils.parseUnits(amount.toString(), decimals))) {
      logWindow.log(`${colorText('Sufficient allowance already exists', COLORS.GRAY)}`);
      return true;
    }
    logWindow.log(`${colorText(`Approving ${amount} tokens for spending...`, COLORS.MAGENTA)}`);
    const amountInWei = ethers.utils.parseUnits(amount.toString(), decimals);
    const tx = await tokenContract.approve(spenderAddress, amountInWei, { gasLimit: 100000 });
    logWindow.log(`${colorText(`Approval transaction sent: ${tx.hash}`, COLORS.GREEN)}`);
    logWindow.log(`${colorText(`Explorer: ${wallet.provider.network.explorer}/tx/${tx.hash}`, COLORS.GRAY)}`);
    const stopSpinner = showSpinner(
      'Waiting for approval confirmation...',
      `Approval confirmed!`,
      60
    );
    await tx.wait();
    stopSpinner();
    logWindow.log(`${colorText('Approval completed', COLORS.CYAN)}`);
    await updateWalletInfo([wallet], wallet.provider.network);
    return true;
  } catch (error) {
    logWindow.log(`${colorText(`Failed to approve token: ${error.message}`, COLORS.RED)}`);
    return false;
  }
}

async function estimateGasFees(provider) {
  try {
    const feeData = await provider.getFeeData();
    return {
      maxFeePerGas: feeData.maxFeePerGas || ethers.utils.parseUnits('50', 'gwei'),
      maxPriorityFeePerGas: feeData.maxPriorityFeePerGas || ethers.utils.parseUnits('2', 'gwei'),
    };
  } catch (error) {
    logWindow.log(`${colorText(`Failed to estimate gas fees, using defaults: ${error.message}`, COLORS.YELLOW)}`);
    return {
      maxFeePerGas: ethers.utils.parseUnits('50', 'gwei'),
      maxPriorityFeePerGas: ethers.utils.parseUnits('2', 'gwei'),
    };
  }
}

async function swapUSDCtoR2USD(wallet, amount, network) {
  try {
    const usdcBalance = await checkBalance(wallet, USDC_ADDRESS);
    logWindow.log(`${colorText(`Current USDC balance: ${usdcBalance}`, COLORS.WHITE)}`);
    if (parseFloat(usdcBalance) < parseFloat(amount)) {
      logWindow.log(`${colorText(`Insufficient USDC balance: ${usdcBalance} USDC < ${amount} USDC`, COLORS.RED)}`);
      return false;
    }
    const approved = await approveToken(wallet, USDC_ADDRESS, USDC_TO_R2USD_CONTRACT, amount);
    if (!approved) return false;
    const usdcContract = new ethers.Contract(USDC_ADDRESS, ERC20_ABI, wallet);
    const decimals = await usdcContract.decimals();
    const amountInWei = ethers.utils.parseUnits(amount.toString(), decimals);
    const data = ethers.utils.hexConcat([
      USDC_TO_R2USD_METHOD_ID,
      ethers.utils.defaultAbiCoder.encode(
        ['address', 'uint256', 'uint256', 'uint256', 'uint256', 'uint256', 'uint256'],
        [wallet.address, amountInWei, 0, 0, 0, 0, 0]
      ),
    ]);
    const gasFees = await estimateGasFees(wallet.provider);
    const tx = {
      to: USDC_TO_R2USD_CONTRACT,
      data: data,
      ...gasFees,
    };
    const gasLimit = await estimateGas(wallet, tx);
    logWindow.log(`${colorText(`Estimated gas limit: ${gasLimit.toString()}`, COLORS.GRAY)}`);
    logWindow.log(`${colorText(`Initiating swap: ${amount} USDC to R2USD on ${network.name}`, COLORS.YELLOW)}`);
    const stopSpinner = showSpinner(
      `Swapping ${amount} USDC to R2USD...`,
      `Swap completed: ${amount} USDC to R2USD`,
      60
    );
    const signedTx = await wallet.sendTransaction({
      ...tx,
      gasLimit,
    });
    logWindow.log(`${colorText(`Transaction sent: ${signedTx.hash}`, COLORS.GREEN)}`);
    logWindow.log(`${colorText(`Explorer: ${network.explorer}/tx/${signedTx.hash}`, COLORS.GRAY)}`);
    await signedTx.wait();
    stopSpinner();
    const newUSDCBalance = await checkBalance(wallet, USDC_ADDRESS);
    const newR2USDBalance = await checkBalance(wallet, R2USD_ADDRESS);
    logWindow.log(`${colorText(`New USDC balance: ${newUSDCBalance}`, COLORS.WHITE)}`);
    logWindow.log(`${colorText(`New R2USD balance: ${newR2USDBalance}`, COLORS.WHITE)}`);
    await updateWalletInfo([wallet], network);
    return true;
  } catch (error) {
    logWindow.log(`${colorText(`Failed to swap USDC to R2USD: ${error.message}`, COLORS.RED)}`);
    if (error.transactionHash) {
      logWindow.log(`${colorText(`Failed transaction: ${network.explorer}/tx/${error.transactionHash}`, COLORS.RED)}`);
    }
    return false;
  }
}

async function swapR2USDtoUSDC(wallet, amount, network) {
  try {
    const r2usdBalance = await checkBalance(wallet, R2USD_ADDRESS);
    logWindow.log(`${colorText(`Current R2USD balance: ${r2usdBalance}`, COLORS.WHITE)}`);
    if (parseFloat(r2usdBalance) < parseFloat(amount)) {
      logWindow.log(`${colorText(`Insufficient R2USD balance: ${r2usdBalance} R2USD < ${amount} R2USD`, COLORS.RED)}`);
      return false;
    }
    const approved = await approveToken(wallet, R2USD_ADDRESS, R2USD_TO_USDC_CONTRACT, amount);
    if (!approved) return false;
    const r2usdContract = new ethers.Contract(R2USD_ADDRESS, ERC20_ABI, wallet);
    const decimals = await r2usdContract.decimals();
    const amountInWei = ethers.utils.parseUnits(amount.toString(), decimals);
    const minOutput = amountInWei.mul(97).div(100);
    logWindow.log(`${colorText(`Swapping ${amount} R2USD, expecting at least ${ethers.utils.formatUnits(minOutput, decimals)} USDC`, COLORS.GRAY)}`);
    const data =
      R2USD_TO_USDC_METHOD_ID +
      '0000000000000000000000000000000000000000000000000000000000000000' +
      '0000000000000000000000000000000000000000000000000000000000000001' +
      amountInWei.toHexString().slice(2).padStart(64, '0') +
      minOutput.toHexString().slice(2).padStart(64, '0');
    const gasFees = await estimateGasFees(wallet.provider);
    const tx = {
      to: R2USD_TO_USDC_CONTRACT,
      data: data,
      ...gasFees,
    };
    const gasLimit = await estimateGas(wallet, tx);
    logWindow.log(`${colorText(`Estimated gas limit: ${gasLimit.toString()}`, COLORS.GRAY)}`);
    logWindow.log(`${colorText(`Initiating swap: ${amount} R2USD to USDC on ${network.name}`, COLORS.YELLOW)}`);
    const stopSpinner = showSpinner(
      `Swapping ${amount} R2USD to USDC...`,
      `Swap completed: ${amount} R2USD to USDC`,
      60
    );
    const signedTx = await wallet.sendTransaction({
      ...tx,
      gasLimit,
    });
    logWindow.log(`${colorText(`Transaction sent: ${signedTx.hash}`, COLORS.GREEN)}`);
    logWindow.log(`${colorText(`Explorer: ${network.explorer}/tx/${signedTx.hash}`, COLORS.GRAY)}`);
    const receipt = await signedTx.wait();
    stopSpinner();
    if (receipt.status === 0) {
      logWindow.log(`${colorText(`Transaction reverted. Check contract state or logs.`, COLORS.RED)}`);
      return false;
    }
    const newUSDCBalance = await checkBalance(wallet, USDC_ADDRESS);
    const newR2USDBalance = await checkBalance(wallet, R2USD_ADDRESS);
    logWindow.log(`${colorText(`New USDC balance: ${newUSDCBalance}`, COLORS.WHITE)}`);
    logWindow.log(`${colorText(`New R2USD balance: ${newR2USDBalance}`, COLORS.WHITE)}`);
    await updateWalletInfo([wallet], network);
    return true;
  } catch (error) {
    logWindow.log(`${colorText(`Failed to swap R2USD to USDC: ${error.message}`, COLORS.RED)}`);
    if (error.transactionHash) {
      logWindow.log(`${colorText(`Failed transaction: ${network.explorer}/tx/${error.transactionHash}`, COLORS.RED)}`);
    }
    return false;
  }
}

async function stakeR2USD(wallet, amount, network) {
  if (network.name !== 'sepolia') {
    logWindow.log(`${colorText(`Staking not supported on ${network.name}`, COLORS.RED)}`);
    return false;
  }
  try {
    const r2usdBalance = await checkBalance(wallet, R2USD_ADDRESS);
    logWindow.log(`${colorText(`Current R2USD balance: ${r2usdBalance}`, COLORS.WHITE)}`);
    if (parseFloat(r2usdBalance) < parseFloat(amount)) {
      logWindow.log(`${colorText(`Insufficient R2USD balance: ${r2usdBalance} R2USD < ${amount} R2USD`, COLORS.RED)}`);
      return false;
    }
    const r2usdContract = new ethers.Contract(R2USD_ADDRESS, ERC20_ABI, wallet);
    const decimals = await r2usdContract.decimals();
    const amountInWei = ethers.utils.parseUnits(amount.toString(), decimals);
    const currentAllowance = await r2usdContract.allowance(wallet.address, STAKE_R2USD_CONTRACT);
    if (currentAllowance.lt(amountInWei)) {
      logWindow.log(`${colorText(`Approving ${amount} R2USD for staking...`, COLORS.MAGENTA)}`);
      const approveTx = await r2usdContract.approve(STAKE_R2USD_CONTRACT, amountInWei, { gasLimit: 100000 });
      logWindow.log(`${colorText(`Approval transaction sent: ${approveTx.hash}`, COLORS.GREEN)}`);
      logWindow.log(`${colorText(`Explorer: ${network.explorer}/tx/${approveTx.hash}`, COLORS.GRAY)}`);
      const stopSpinner = showSpinner(
        'Waiting for approval confirmation...',
        `Approval confirmed!`,
        60
      );
      await approveTx.wait();
      stopSpinner();
      logWindow.log(`${colorText('Approval completed', COLORS.CYAN)}`);
    } else {
      logWindow.log(`${colorText('Sufficient allowance already exists', COLORS.GRAY)}`);
    }
    const data =
      STAKE_R2USD_METHOD_ID +
      amountInWei.toHexString().slice(2).padStart(64, '0') +
      '0'.repeat(576);
    const gasFees = await estimateGasFees(wallet.provider);
    const tx = {
      to: STAKE_R2USD_CONTRACT,
      data: data,
      ...gasFees,
    };
    const gasLimit = await estimateGas(wallet, tx);
    logWindow.log(`${colorText(`Estimated gas limit: ${gasLimit.toString()}`, COLORS.GRAY)}`);
    logWindow.log(`${colorText(`Initiating staking: ${amount} R2USD to sR2USD`, COLORS.YELLOW)}`);
    const stopSpinner = showSpinner(
      `Staking ${amount} R2USD to sR2USD...`,
      `Staking completed: ${amount} R2USD to sR2USD`,
      60
    );
    const signedTx = await wallet.sendTransaction({
      ...tx,
      gasLimit,
    });
    logWindow.log(`${colorText(`Transaction sent: ${signedTx.hash}`, COLORS.GREEN)}`);
    logWindow.log(`${colorText(`Explorer: ${network.explorer}/tx/${signedTx.hash}`, COLORS.GRAY)}`);
    const receipt = await signedTx.wait();
    stopSpinner();
    if (receipt.status === 0) {
      logWindow.log(`${colorText(`Transaction reverted. Check contract state or logs.`, COLORS.RED)}`);
      return false;
    }
    const newR2USDBalance = await checkBalance(wallet, R2USD_ADDRESS);
    const newSR2USDBalance = await checkBalance(wallet, SR2USD_ADDRESS);
    logWindow.log(`${colorText(`New R2USD balance: ${newR2USDBalance}`, COLORS.WHITE)}`);
    logWindow.log(`${colorText(`New sR2USD balance: ${newSR2USDBalance}`, COLORS.WHITE)}`);
    await updateWalletInfo([wallet], network);
    return true;
  } catch (error) {
    logWindow.log(`${colorText(`Failed to stake R2USD: ${error.message}`, COLORS.RED)}`);
    if (error.transactionHash) {
      logWindow.log(`${colorText(`Failed transaction: ${network.explorer}/tx/${error.transactionHash}`, COLORS.RED)}`);
    }
    return false;
  }
}

// UI helper functions
function showSpinner(message, completionMessage = 'Done!', duration = 60) {
  const spinnerStyles = [
    ['⠋', '⠙', '⠹', '⠸', '⠼', '⠴', '⠦', '⠧', '⠇', '⠏'], // Dots
    ['-', '=', '≡'], // Bars
    ['⣾', '⣽', '⣻', '⢿', '⡿', '⣟', '⣯', '⣷'], // Brail
  ];
  const spinner = spinnerStyles[Math.floor(Math.random() * spinnerStyles.length)];
  let i = 0;
  logWindow.log(`${colorText(`${message} ${spinner[0]}`, COLORS.YELLOW)}`);
  const logIndex = logWindow.getLines().length - 1;
  const interval = setInterval(() => {
    logWindow.setLine(logIndex, `${colorText(`${message} ${spinner[i++ % spinner.length]}`, COLORS.YELLOW)}`);
    screen.render();
  }, duration);
  return () => {
    clearInterval(interval);
    logWindow.setLine(logIndex, `${colorText(completionMessage, COLORS.GREEN)}`);
    screen.render();
  };
}

function getInput(promptText) {
  return new Promise((resolve) => {
    logWindow.log(`${colorText(promptText, COLORS.YELLOW)}`);
    inputBox.setValue('');
    inputBox.show();
    screen.render();
    inputBox.once('submit', (value) => {
      inputBox.hide();
      screen.render();
      resolve(value);
    });
    inputBox.focus();
    inputBox.readInput();
  });
}

function showBanner() {
  const banner = [
    '>>> SYSTEM BOOT INITIATED',
    '[[ R2 AUTO BOT ]] - BY KAZUHA',
    '----------------------------------',
  ];
  banner.forEach((line, index) => {
    setTimeout(() => {
      logWindow.log(`${colorText(line, index === 1 ? COLORS.CYAN : COLORS.GREEN)}`);
      screen.render();
    }, index * 150);
  });
}

async function showMenu(wallets) {
  const menuItems = [
    `1. Swaps and Staking (Manual)`,
    `2. Auto Run All`,
    `3. Exit`,
  ];
  logWindow.log(`${colorText('========== R2 AUTO BOT MENU ==========', COLORS.WHITE)}`);
  for (const item of menuItems) {
    logWindow.log(`${colorText(item, COLORS.YELLOW)}`);
    await new Promise(resolve => setTimeout(resolve, 100));
    screen.render();
  }
  logWindow.log(`${colorText('=====================================', COLORS.WHITE)}`);

  const option = await getInput('Select an option (1-3): ');
  switch (option) {
    case '1':
      await handleSwapsAndStaking(wallets);
      break;
    case '2':
      await handleAutoRunAll(wallets);
      break;
    case '3':
      logWindow.log(`${colorText(`Exiting application...`, COLORS.GRAY)}`);
      process.exit(0);
    default:
      logWindow.log(`${colorText('Invalid option. Please select 1-3.', COLORS.YELLOW)}`);
      await showMenu(wallets);
      break;
  }
}

async function selectWallet(wallets) {
  if (wallets.length === 1) {
    logWindow.log(`${colorText(`Using wallet: ${wallets[0].wallet.address.slice(0, 6)}...${wallets[0].wallet.address.slice(-4)}`, COLORS.WHITE)}`);
    return wallets[0];
  }
  logWindow.log(`${colorText('Available wallets:', COLORS.WHITE)}`);
  wallets.forEach((walletObj, index) => {
    logWindow.log(`${colorText(`${index + 1}. ${walletObj.wallet.address.slice(0, 6)}...${walletObj.wallet.address.slice(-4)}`, COLORS.YELLOW)}`);
  });
  const input = await getInput('Select wallet number (or "all" for all wallets): ');
  if (input.toLowerCase() === 'all') {
    return wallets;
  }
  const index = parseInt(input) - 1;
  if (isNaN(index) || index < 0 || index >= wallets.length) {
    logWindow.log(`${colorText('Invalid selection. Using first wallet.', COLORS.YELLOW)}`);
    return wallets[0];
  }
  logWindow.log(`${colorText(`Using wallet: ${wallets[index].wallet.address.slice(0, 6)}...${wallets[index].wallet.address.slice(-4)}`, COLORS.WHITE)}`);
  return wallets[index];
}

async function selectNetwork() {
  if (NETWORKS.length === 1) {
    logWindow.log(`${colorText(`Using network: ${NETWORKS[0].name}`, COLORS.WHITE)}`);
    return [NETWORKS[0]];
  }
  logWindow.log(`${colorText('Available networks:', COLORS.WHITE)}`);
  NETWORKS.forEach((network, index) => {
    logWindow.log(`${colorText(`${index + 1}. ${network.name}`, COLORS.YELLOW)}`);
  });
  const input = await getInput('Select network number (or "all" for all networks): ');
  if (input.toLowerCase() === 'all') {
    return NETWORKS;
  }
  const index = parseInt(input) - 1;
  if (isNaN(index) || index < 0 || index >= NETWORKS.length) {
    logWindow.log(`${colorText('Invalid selection. Using first network.', COLORS.YELLOW)}`);
    return [NETWORKS[0]];
  }
  logWindow.log(`${colorText(`Using network: ${NETWORKS[index].name}`, COLORS.WHITE)}`);
  return [NETWORKS[index]];
}

async function handleSwapsAndStaking(wallets) {
  try {
    const selectedWallets = wallets.length === 1 ? wallets[0] : await selectWallet(wallets);
    const isAllWallets = Array.isArray(selectedWallets);
    const walletList = isAllWallets ? selectedWallets : [selectedWallets];

    const selectedNetworks = await selectNetwork();
    const isAllNetworks = selectedNetworks.length > 1;

    // Popup for amount and number of transactions
    logWindow.log(`${colorText('=== Swap/Staking Configuration ===', COLORS.CYAN)}`);
    const amount = await getInput('Enter amount for swaps/staking (or "back" to return): ');
    if (amount.toLowerCase() === 'back') {
      await showMenu(wallets);
      return;
    }
    const parsedAmount = parseFloat(amount);
    if (isNaN(parsedAmount) || parsedAmount <= 0) {
      logWindow.log(`${colorText('Invalid amount. Enter a positive number.', COLORS.RED)}`);
      await handleSwapsAndStaking(wallets);
      return;
    }
    const numTxs = await getInput('Enter number of transactions (or "back" to return): ');
    if (numTxs.toLowerCase() === 'back') {
      await showMenu(wallets);
      return;
    }
    const parsedNumTxs = parseInt(numTxs);
    if (isNaN(parsedNumTxs) || parsedNumTxs <= 0) {
      logWindow.log(`${colorText('Invalid number. Enter a positive integer.', COLORS.RED)}`);
      await handleSwapsAndStaking(wallets);
      return;
    }
    logWindow.log(`${colorText('=================================', COLORS.CYAN)}`);

    const runSwapCycle = async () => {
      for (const network of selectedNetworks) {
        logWindow.log(`${colorText(`Processing network: ${network.name}`, COLORS.CYAN)}`);
        for (const walletObj of walletList) {
          // Initialize wallet for the current network if not already initialized
          let wallet = walletObj.wallet;
          if (walletObj.network.name !== network.name) {
            const result = await initializeWallet(walletObj.privateKey, network);
            wallet = result.wallet;
          }
          logWindow.log(`${colorText(`Processing wallet: ${wallet.address.slice(0, 6)}...${wallet.address.slice(-4)} on ${network.name}`, COLORS.WHITE)}`);

          // USDC to R2USD Swaps
          logWindow.log(`${colorText(`Starting ${parsedNumTxs} USDC to R2USD swaps`, COLORS.CYAN)}`);
          for (let i = 1; i <= parsedNumTxs; i++) {
            logWindow.log(`${colorText(`Swap ${i}/${parsedNumTxs}: ${parsedAmount} USDC to R2USD`, COLORS.YELLOW)}`);
            const success = await swapUSDCtoR2USD(wallet, parsedAmount, network);
            logWindow.log(
              `${colorText(`Swap ${i} ${success ? 'completed!' : 'failed.'}`, success ? COLORS.GREEN : COLORS.RED)}`
            );
            if (!success) break;
            await new Promise(resolve => setTimeout(resolve, 1000)); // 1s delay
          }

          // R2USD to USDC Swaps
          logWindow.log(`${colorText(`Starting ${parsedNumTxs} R2USD to USDC swaps`, COLORS.CYAN)}`);
          for (let i = 1; i <= parsedNumTxs; i++) {
            logWindow.log(`${colorText(`Swap ${i}/${parsedNumTxs}: ${parsedAmount} R2USD to USDC`, COLORS.YELLOW)}`);
            const success = await swapR2USDtoUSDC(wallet, parsedAmount, network);
            logWindow.log(
              `${colorText(`Swap ${i} ${success ? 'completed!' : 'failed.'}`, success ? COLORS.GREEN : COLORS.RED)}`
            );
            if (!success) break;
            await new Promise(resolve => setTimeout(resolve, 1000)); // 1s delay
          }

          // R2USD to sR2USD Staking (only for Sepolia)
          if (network.name === 'sepolia') {
            logWindow.log(`${colorText(`Starting ${parsedNumTxs} R2USD to sR2USD stakes`, COLORS.CYAN)}`);
            for (let i = 1; i <= parsedNumTxs; i++) {
              logWindow.log(`${colorText(`Stake ${i}/${parsedNumTxs}: ${parsedAmount} R2USD to sR2USD`, COLORS.YELLOW)}`);
              const success = await stakeR2USD(wallet, parsedAmount, network);
              logWindow.log(
                `${colorText(`Stake ${i} ${success ? 'completed!' : 'failed.'}`, success ? COLORS.GREEN : COLORS.RED)}`
              );
              if (!success) break;
              await new Promise(resolve => setTimeout(resolve, 1000)); // 1s delay
            }
          }

          logWindow.log(`${colorText(`Completed all tasks for ${wallet.address.slice(0, 6)}...${wallet.address.slice(-4)} on ${network.name}`, COLORS.GREEN)}`);
        }
      }

      logWindow.log(`${colorText(`All swaps/stakes completed! Pausing for 24 hours...`, COLORS.YELLOW)}`);
      await new Promise(resolve => setTimeout(resolve, 24 * 60 * 60 * 1000)); // 24-hour delay
      logWindow.log(`${colorText(`Restarting swap cycle...`, COLORS.CYAN)}`);
      await handleSwapsAndStaking(wallets); // Re-prompt for new amount and numTxs
    };

    await runSwapCycle();
  } catch (error) {
    logWindow.log(`${colorText(`Error in swap/staking cycle: ${error.message}`, COLORS.RED)}`);
    await showMenu(wallets);
  }
}

async function handleAutoRunAll(wallets) {
  try {
    // Prompt for amount and number of transactions
    logWindow.log(`${colorText('=== Auto Run All Configuration ===', COLORS.CYAN)}`);
    const amount = await getInput('Enter amount for swaps/staking: ');
    const parsedAmount = parseFloat(amount);
    if (isNaN(parsedAmount) || parsedAmount <= 0) {
      logWindow.log(`${colorText('Invalid amount. Enter a positive number.', COLORS.RED)}`);
      await showMenu(wallets);
      return;
    }
    const numTxs = await getInput('Enter number of transactions: ');
    const parsedNumTxs = parseInt(numTxs);
    if (isNaN(parsedNumTxs) || parsedNumTxs <= 0) {
      logWindow.log(`${colorText('Invalid number. Enter a positive integer.', COLORS.RED)}`);
      await showMenu(wallets);
      return;
    }
    logWindow.log(`${colorText('=================================', COLORS.CYAN)}`);

    const runAutoCycle = async () => {
      // Process networks in order: sepolia, plume_testnet, arbitrum_sepolia
      for (const network of NETWORKS) {
        logWindow.log(`${colorText(`Processing network: ${network.name}`, COLORS.CYAN)}`);
        for (const walletObj of wallets) {
          // Initialize wallet for the current network
          let wallet;
          try {
            const result = await initializeWallet(walletObj.privateKey, network);
            wallet = result.wallet;
          } catch (error) {
            logWindow.log(`${colorText(`Skipping wallet ${walletObj.wallet.address.slice(0, 6)}... on ${network.name} due to initialization error`, COLORS.RED)}`);
            continue;
          }
          logWindow.log(`${colorText(`Processing wallet: ${wallet.address.slice(0, 6)}...${wallet.address.slice(-4)} on ${network.name}`, COLORS.WHITE)}`);

          // USDC to R2USD Swaps
          logWindow.log(`${colorText(`Starting ${parsedNumTxs} USDC to R2USD swaps`, COLORS.CYAN)}`);
          for (let i = 1; i <= parsedNumTxs; i++) {
            logWindow.log(`${colorText(`Swap ${i}/${parsedNumTxs}: ${parsedAmount} USDC to R2USD`, COLORS.YELLOW)}`);
            const success = await swapUSDCtoR2USD(wallet, parsedAmount, network);
            logWindow.log(
              `${colorText(`Swap ${i} ${success ? 'completed!' : 'failed.'}`, success ? COLORS.GREEN : COLORS.RED)}`
            );
            if (!success) break;
            await new Promise(resolve => setTimeout(resolve, 1000)); // 1s delay
          }

          // R2USD to USDC Swaps
          logWindow.log(`${colorText(`Starting ${parsedNumTxs} R2USD to USDC swaps`, COLORS.CYAN)}`);
          for (let i = 1; i <= parsedNumTxs; i++) {
            logWindow.log(`${colorText(`Swap ${i}/${parsedNumTxs}: ${parsedAmount} R2USD to USDC`, COLORS.YELLOW)}`);
            const success = await swapR2USDtoUSDC(wallet, parsedAmount, network);
            logWindow.log(
              `${colorText(`Swap ${i} ${success ? 'completed!' : 'failed.'}`, success ? COLORS.GREEN : COLORS.RED)}`
            );
            if (!success) break;
            await new Promise(resolve => setTimeout(resolve, 1000)); // 1s delay
          }

          // R2USD to sR2USD Staking (only for Sepolia)
          if (network.name === 'sepolia') {
            logWindow.log(`${colorText(`Starting ${parsedNumTxs} R2USD to sR2USD stakes`, COLORS.CYAN)}`);
            for (let i = 1; i <= parsedNumTxs; i++) {
              logWindow.log(`${colorText(`Stake ${i}/${parsedNumTxs}: ${parsedAmount} R2USD to sR2USD`, COLORS.YELLOW)}`);
              const success = await stakeR2USD(wallet, parsedAmount, network);
              logWindow.log(
                `${colorText(`Stake ${i} ${success ? 'completed!' : 'failed.'}`, success ? COLORS.GREEN : COLORS.RED)}`
              );
              if (!success) break;
              await new Promise(resolve => setTimeout(resolve, 1000)); // 1s delay
            }
          }

          logWindow.log(`${colorText(`Completed all tasks for ${wallet.address.slice(0, 6)}...${wallet.address.slice(-4)} on ${network.name}`, COLORS.GREEN)}`);
        }
      }

      logWindow.log(`${colorText(`All networks processed! Pausing for 24 hours...`, COLORS.YELLOW)}`);
      await new Promise(resolve => setTimeout(resolve, 24 * 60 * 60 * 1000)); // 24-hour delay
      logWindow.log(`${colorText(`Restarting auto-run cycle...`, COLORS.CYAN)}`);
      await runAutoCycle(); // Restart the cycle
    };

    await runAutoCycle();
  } catch (error) {
    logWindow.log(`${colorText(`Error in auto-run cycle: ${error.message}`, COLORS.RED)}`);
    await showMenu(wallets);
  }
}

async function main() {
  try {
    // Show banner first
    await new Promise(resolve => {
      showBanner();
      setTimeout(resolve, 450); // Wait for banner animation (3 lines * 150ms)
    });

    loadProxies();
    loadPrivateKeys();
    const wallets = [];
    // Initialize each private key only once for the first network initially
    for (const privateKey of privateKeys) {
      try {
        const result = await initializeWallet(privateKey, NETWORKS[0]);
        wallets.push({ privateKey, wallet: result.wallet, network: NETWORKS[0] });
      } catch (error) {
        // Skip failed wallet initialization
      }
    }
    if (wallets.length === 0) {
      logWindow.log(`${colorText(`No valid wallets initialized. Exiting.`, COLORS.RED)}`);
      process.exit(1);
    }
    // Update info for the first network's wallets initially
    await updateWalletInfo(wallets.map(w => w.wallet), NETWORKS[0]);

    // Show menu immediately
    await showMenu(wallets);
  } catch (error) {
    logWindow.log(`${colorText(`Fatal error: ${error.message}`, COLORS.RED)}`);
    process.exit(1);
  }
}

main();
