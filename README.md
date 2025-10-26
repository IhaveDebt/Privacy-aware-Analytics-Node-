
---

# 3 — Privacy-aware Analytics (Node)  
**What:** Small CLI demo of Laplace differential privacy for sums/means + naive privacy budget accountant.

**File:** `src/dp_analytics.js`

```javascript
#!/usr/bin/env node
/**
 * Differentially-private Analytics Demo (Node)
 *
 * Usage:
 *   node src/dp_analytics.js
 *
 * Educational implementation of Laplace mechanism and simple privacy accountant.
 */

function laplace(scale){
  // sample from Laplace(0, scale)
  const u = Math.random() - 0.5;
  return -scale * Math.sign(u) * Math.log(1 - 2 * Math.abs(u));
}

class PrivacyAccountant {
  constructor(totalEps){ this.total = totalEps; this.used = 0; }
  charge(eps){
    if (this.used + eps > this.total) throw new Error('Privacy budget exceeded');
    this.used += eps;
  }
  remaining(){ return this.total - this.used; }
}

class DPAggregator {
  constructor(accountant, sensitivity = 1.0){
    this.accountant = accountant;
    this.sensitivity = sensitivity;
  }
  dpSum(values, eps){
    this.accountant.charge(eps);
    const trueSum = values.reduce((a,b)=>a+b,0);
    const scale = this.sensitivity / eps;
    return trueSum + laplace(scale);
  }
  dpMean(values, eps){
    if (values.length === 0) return null;
    // split budget heuristically between sum and count
    const epsSum = eps * 0.7;
    const epsCount = eps * 0.3;
    this.accountant.charge(epsSum);
    const noisySum = values.reduce((a,b)=>a+b,0) + laplace(this.sensitivity / epsSum);
    this.accountant.charge(epsCount);
    const noisyCount = values.length + laplace(1.0 / epsCount);
    return noisySum / Math.max(1, noisyCount);
  }
}

// Demo
if (require.main === module){
  const acct = new PrivacyAccountant(1.0);
  const dp = new DPAggregator(acct);
  const data = Array.from({length:200},()=>Math.random()*100);
  console.log("Remaining budget:", acct.remaining());
  console.log("True sum:", data.reduce((a,b)=>a+b,0).toFixed(3));
  console.log("DP sum (eps=0.5):", dp.dpSum(data, 0.5).toFixed(3));
  console.log("Remaining budget:", acct.remaining().toFixed(3));
}
module.exports = { PrivacyAccountant, DPAggregator };
