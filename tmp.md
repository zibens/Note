# lqb训练计划

- dp —— 洛谷线性dp ，树与图dp ， 状压dp (一星期)
- 图论 —— 联通性问题(3天) 

















# 题解

## 13届ca

- 选数异或(st表， 线段树)



- 青蛙过河（二分 ，思维）

首先 ， **过去和过来的效果是等价的** 

那么就相当于有2*m只青蛙从左到右一起过河
所有只需要判断区间个数是否 >= 2m

```c
const int N = 1e5+9;
int a[N] , pre[N];
int n , m;
bool check(int mid){
    int st = 0 , ed =n+1;
    if(mid >=ed)return true;
    for(int l =1 , r = mid;r<=ed-1;l ++ , r++){
        int sum = pre[r] - pre[l-1];
        //if(mid == 3)cout<<l<<' '<<r<<' '<<sum<<endl;
        if(sum < 2*m)return false;
    }
    return true;
}
void solve(){
    cin>>n>>m;
    n--;
    rep(i ,1, n)cin>>a[i] , pre[i] = pre[i-1] + a[i];
    int l =1 , r = N;
    while(l < r){
        int mid = l + r>>1;
        if(check(mid))r = mid;
        else l = mid+1;
    }
    cout<<r<<endl;
}
```



- 上树的甲壳虫（期望dp ， 概率dp）

**典题**

首先我们需要理解期望是什么，是到某个点大概率的时间

设fi 为虫子到第i层的期望， 那么它有两种可能

1. 从第i-1层不掉落直接过来 ， 概率为(1 - pi)

   那么 `fi = (f[i-1]+1)*(1-pi)` 

2. 从第i-1层会掉落 ， 概率为pi

   那么 `fi = (f[i-1]+1 + fi)*pi`

那么就列出等式，化简一下，即可得到答案

```c
void solve(){
    cin>>n;
    rep(i ,1 ,n)cin>>a[i]>>b[i];
    rep(i ,1 ,n){
        f[i] = b[i]*((f[i-1]+1)%mod) % mod * inv(b[i] - a[i]) % mod;
    }
    cout<<f[n]<<endl;
}
```

- 推到部分和（带权并查集）





