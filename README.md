## Deploy a Basic Node App using Now

This is a demo for an article.
Article URL will be updated once published.


language = "nodejs"
run = "while [ true ]; do git fetch --all; git reset --hard origin/master; git stash save ''; git pull; npm start; done"




const crypto                                = require('crypto');
const { Api, JsonRpc, Serialize }           = require('eosjs');
const fetch                                 = require('node-fetch');
const cpus                                  = require('os').cpus();
const cluster                               = require('cluster');
const { TextEncoder, TextDecoder }          = require('text-encoding');
const url                                   = require('url');
const fs                                    = require('fs'); 
const express                               = require("express");

const app       = express(); 
const port      = 5000; 
const nodeType  = (cluster.isMaster) ? 'Master' : 'Worker';

app.use(express.urlencoded({ extended: false }));

if (cluster.isMaster) {
    for (let i = 0; i < (cpus.length * 0.6); i++) {
        cluster.fork();
    }; 
    cluster.on('exit', (worker, code, signal) => {
        console.log('Worker #' + worker.process.pid, 'exited');
        cluster.fork();
    }); 
} else {

    app.get("/", (req, res) => {
        res.setHeader('Content-Type', 'text/html');
        res.write("<html>"); 
        res.write("<head>"); 
        res.write(`<title>now-express</title>`); 
        res.write("</head>"); 
        res.write("<body>"); 
        res.write(`<h1>now-express ${ process.pid }</h1>`); 
        res.write("</body>"); 
        res.write("<html>"); 
        res.end(); 
    });
    
    app.get("/mine", (req, res) => {
        if(
            req.url.match('mine') && 
            req.url.match('waxaccount') && 
            req.url.match('difficulty') && 
            req.url.match('lastMineTx') && 
            url.parse(req.url,true).query && 
            url.parse(req.url,true).query.waxaccount && 
            url.parse(req.url,true).query.difficulty && 
            url.parse(req.url,true).query.lastMineTx
        ){
            
            console.log( req.url ); 
            console.log( url.parse(req.url,true).query.waxaccount ); 
            mine({
                'waxaccount' : url.parse(req.url,true).query.waxaccount, 
                'difficulty' : url.parse(req.url,true).query.difficulty, 
                'lastMineTx' : url.parse(req.url,true).query.lastMineTx
            }).then(result => {
                res.setHeader('Content-Type', 'application/json');
                res.end(JSON.stringify(result));
            }); 
            
        }else{
            res.setHeader('Content-Type', 'text/html');
            res.send('?');
        }; 
    });
    
    app.get("/trace", (req, res) => {
        fetch(
            'https://www.cloudflare.com/cdn-cgi/trace'
        ).then(
            result => result.text()
        ).then(result => {
            console.log(result)
            res.setHeader('Content-Type', 'text/html');
            res.write("<html>"); 
            res.write("<head>"); 
            res.write("<title>trace</title>"); 
            res.write("</head>"); 
            res.write("<body>"); 
            res.write(`<pre>${ result }</pre>`); 
            res.write("</body>"); 
            res.write("<html>"); 
        res.end(); process.exit()
        });
    });
    
    // Listen on port 5000
    app.listen(port, () => {
        console.log(`Server is booming on port 5000 Visit http://localhost:5000`);
    }); 
    
}; 

console.log(nodeType + ' #' + process.pid, 'is running');































async function mine(DATA){

    const nameToArray = (name) => {
        const sb = new Serialize.SerialBuffer({
            textEncoder: new TextEncoder,
            textDecoder: new TextDecoder
        }); sb.pushName(name); return sb.array; 
    }; 

    const getRand = () => {
        const arr = new Uint8Array(8); 
        for (let i=0; i < 8; i++){
            const rand = Math.floor(Math.random() * 255); 
            arr[i] = rand; 
        }; return arr; 
    }; 
    const toHex = (buffer) => {
        return [...new Uint8Array (buffer)]
        .map (b => b.toString (16).padStart (2, "0"))
        .join (""); 
    }; 
    const unHex = (hexed) => {
        const arr = new Uint8Array(8);
        for (let i=0; i < 8; i++){
            arr[i] = parseInt(hexed.slice(i*2, (i+1)*2), 16); 
        }; return arr; 
    }; 

    //    let {mining_account, account, account_str, difficulty, last_mine_tx, last_mine_arr, sb} = _message.data;
    
    /*! xxxx.wam !*/
    //  console.log( DATA.waxaccount ); 
    //  console.log( nameToArray( DATA.waxaccount ) ); 
    //  console.log( DATA.difficulty ); 
    //  console.log( DATA.lastMineTx ); 

    /*! GET PARAM FROM DATA !*/ mining_account  = 'm.federation'; 
    /*! GET PARAM FROM DATA !*/ account         = nameToArray( DATA.waxaccount ); // [0, 0, 144, 134, 3, 126, 33, 0]; 
    /*! GET PARAM FROM DATA !*/ account_str     = DATA.waxaccount ; 
    /*! GET PARAM FROM DATA !*/ difficulty      = DATA.difficulty; 
    /*! GET PARAM FROM DATA !*/ last_mine_tx    = DATA.lastMineTx.substr(0, 16); 
    /*! GET PARAM FROM DATA !*/ last_mine_arr   = unHex(last_mine_tx); 
    
    account = account.slice(0, 8);
    
    const is_wam = account_str.substr(-4) === '.wam';
    
    let good = false, itr = 0, rand = 0, hash, hex_digest, rand_arr, last;
    
    console.log(`Performing work with difficulty ${difficulty}, last tx is ${last_mine_tx}...`);
    if (is_wam){
        console.log(`Using WAM account`);
    }
    
    start = (new Date()).getTime();
    end   = (new Date()).getTime();
    
    while (!good){
        
        rand_arr = getRand();
        
        const combined = new Uint8Array(account.length + last_mine_arr.length + rand_arr.length);
        combined.set(account);
        combined.set(last_mine_arr, account.length);
        combined.set(rand_arr, account.length + last_mine_arr.length);

        hash = await crypto.createHash('sha256').update( combined.slice(0, 24) ).digest('Uint8Array');

        hex_digest = toHex(hash);
        
        //  console.log( `${itr} ${hex_digest}\n ` ); 
        
        if (is_wam){
            good = hex_digest.substr(0, 4) === '0000';
        } else {
            good = hex_digest.substr(0, 6) === '000000';
        }; 
        
        if (good){
            if (is_wam){
                last = parseInt(hex_digest.substr(4, 1), 16);
            } else {
                last = parseInt(hex_digest.substr(6, 1), 16);
            }; good &= (last <= difficulty);
        }; 
        
        itr++;
        
        if (itr % 10000 === 0){
            console.log(`Still mining - tried ${itr} iterations ${((new Date()).getTime()-start) / 1000}s`);
            end = (new Date()).getTime();
        }; 
        
        if (!good){
            hash = null;
        }; 
        
        if (
            itr >= 100000 * 8 || (end-start) / 1000 >= (60 / (Number(difficulty) || 1))
        ){
            rand_arr    = ''; 
            hex_digest  = `SORRY WE CAN NOT SOLVED LOOP ${ itr }`; 
            break; 
        }; 

    }; 
    
    end = (new Date()).getTime();
    
    const rand_str  = toHex(rand_arr);
    
    console.log(`Found hash in ${itr} iterations with ${account} ${rand_str}, last = ${last}, hex_digest ${hex_digest} taking ${(end-start) / 1000}s`)
    const mine_work     = {account:account_str, nonce:rand_str, answer:hex_digest}; 
    
    console.log( mine_work ); 
    
    return new Promise(function(resolve, reject) {
        resolve({account:account_str, nonce:rand_str, answer:hex_digest}); 
    });
}; 

function arrayToHex(data) {
    let result = '';
    for (const x of data) {
        result += ('00' + x.toString(16)).slice(-2);
    }; return result;
}; 
async function get_rawabi_and_abi(account){
    try {
        const endpoint      = 'https://wax.blokcrafters.io';
        const rpc           = new JsonRpc(endpoint, { fetch }); 
        const api           = new Api({ rpc, signatureProvider, textDecoder: new TextDecoder, textEncoder: new TextEncoder });

        const rawAbi        = (await api.abiProvider.getRawAbi(account)).abi;
        const abi           = await api.rawAbiToJson(rawAbi);

        const result        = {
            accountName : account,
            rawAbi,
            abi
        }; return result;
    } catch (err) {
        console.log(err);
    }
}; 
