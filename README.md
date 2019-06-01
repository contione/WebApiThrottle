# WebApiThrottle
   ASP.NET WebApi实现请求频率限制
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method)]
    public class ThrottleAttribute : ActionFilterAttribute
    {
        private readonly HandleRequest _handleRequest;
        public int Seconds { get; set; }
        public ThrottleAttribute(int seconds)
        {
            this._handleRequest = new HandleRequest();
            this.Seconds = seconds;
        }
          public override async Task OnActionExecutingAsync(HttpActionContext actionContext, CancellationToken cancellationToken)
    {
      
        var valid = await this._handleRequest.IsValidRequest(actionContext, Seconds);
        if (!valid)
           actionContext.Response = new HttpResponseMessage((HttpStatusCode)429) { ReasonPhrase = "Too Many Requests!" };
    }
}

  public class HandleRequest
    {
        private string Name  => "Client1";

        private int Multiple => 1000;//2;

        public async Task<bool> IsValidRequest(HttpActionContext actionContext, int Seconds)
        {
            var ControllerName = actionContext.ControllerContext.ControllerDescriptor.ControllerName;
            var ActionName = actionContext.ActionDescriptor.ActionName;
            var allowExecute = false;
            await Task.Factory.StartNew(() =>
            {
                var key = (string.Concat(Name, "-", GetClientIp(actionContext.Request), ControllerName, ActionName).GetHashCode()).ToString();//Math.Abs
                //var result = $"{JsonConvert.SerializeObject(HttpRuntime.Cache)}";
                if (HttpRuntime.Cache[key] == null)
                {
                    HttpRuntime.Cache.Add(key,
                        true, //这是我们可以拥有的最小数据吗？
                        null, // 没有依赖关系
                        DateTime.Now.AddMilliseconds(Seconds * Multiple), // 绝对过期
                        Cache.NoSlidingExpiration,
                        CacheItemPriority.Low,
                        null); //没有回调
                    allowExecute = true;
                }
            });
            return allowExecute;
        }

        private string GetClientIp(HttpRequestMessage request)
        {
            //获取传统context
            if (request.Properties.ContainsKey("MS_HttpContext"))
            {
                //HttpContextWrapper 类是从 HttpContextBase 类派生的。 HttpContextWrapper 类用作 HttpContext 类的包装。 在运行时，通常使用 HttpContextWrapper 类的实例调用 HttpContext 对象上的成员。
                return ((HttpContextWrapper)request.Properties["MS_HttpContext"]).Request.UserHostAddress;
            }
            //CS客户端获取ip地址
            //让与发送消息的远程终结点有关的客户端 IP 地址和端口号可用。
            if (request.Properties.ContainsKey(RemoteEndpointMessageProperty.Name))
            {
                RemoteEndpointMessageProperty prop;
                prop = (RemoteEndpointMessageProperty)request.Properties[RemoteEndpointMessageProperty.Name];
                return prop.Address;
            }
            return null;
        }
    }
