apm-ser: 服务端与客户端通信介绍.

#### 1.职责

#### 2.ser
1.接受post 请求， 拿到压缩的json数据
2.解码. queue.push.   ==> publish 发送.

#### 3.cli
1.生成json.
2.压缩zlib/gzip
3.http.post ==> http://localhost:8200/v1/transactions. data 

#### ser
```go
package decoder

import (
   "compress/gzip"
   "compress/zlib"
   "encoding/json"
   "fmt"
   "net/http"

   "io/ioutil"
   "strings"

   "github.com/pkg/errors"

   "github.com/elastic/apm-server/utility"
)

type Decoder func(req *http.Request) (map[string]interface{}, error)

func DecodeLimitJSONData(maxSize int64) Decoder {
   return func(req *http.Request) (map[string]interface{}, error) {
      contentType := req.Header.Get("Content-Type")
      if !strings.Contains(contentType, "application/json") {
         return nil, fmt.Errorf("invalid content type: %s", req.Header.Get("Content-Type"))
      }

      reader := req.Body
      if reader == nil {
         return nil, errors.New("no content")
      }

      switch req.Header.Get("Content-Encoding") {
      case "deflate":
         var err error
         reader, err = zlib.NewReader(reader)
         if err != nil {
            return nil, err
         }

      case "gzip":
         var err error
         reader, err = gzip.NewReader(reader)
         if err != nil {
            return nil, err
         }
      }
      v := make(map[string]interface{})
      if err := json.NewDecoder(http.MaxBytesReader(nil, reader, maxSize)).Decode(&v); err != nil {
         // If we run out of memory, for example
         return nil, errors.Wrap(err, "data read error")
      }
      return v, nil
   }
}

func DecodeSourcemapFormData(req *http.Request) (map[string]interface{}, error) {
   contentType := req.Header.Get("Content-Type")
   if !strings.Contains(contentType, "multipart/form-data") {
      return nil, fmt.Errorf("invalid content type: %s", req.Header.Get("Content-Type"))
   }

   file, _, err := req.FormFile("sourcemap")
   if err != nil {
      return nil, err
   }
   defer file.Close()

   sourcemapBytes, err := ioutil.ReadAll(file)
   if err != nil {
      return nil, err
   }

   payload := map[string]interface{}{
      "sourcemap":       string(sourcemapBytes),
      "service_name":    req.FormValue("service_name"),
      "service_version": req.FormValue("service_version"),
      "bundle_filepath": utility.CleanUrlPath(req.FormValue("bundle_filepath")),
   }

   return payload, nil
}

func DecodeUserData(decoder Decoder, enabled bool) Decoder {
   if !enabled {
      return decoder
   }

   augment := func(req *http.Request) map[string]interface{} {
      return map[string]interface{}{
         "ip":         utility.ExtractIP(req),
         "user_agent": req.Header.Get("User-Agent"),
      }
   }
   return augmentData(decoder, "user", augment)
}

func DecodeSystemData(decoder Decoder, enabled bool) Decoder {
   if !enabled {
      return decoder
   }

   augment := func(req *http.Request) map[string]interface{} {
      return map[string]interface{}{"ip": utility.ExtractIP(req)}
   }
   return augmentData(decoder, "system", augment)
}

func augmentData(decoder Decoder, key string, augment func(req *http.Request) map[string]interface{}) Decoder {
   return func(req *http.Request) (map[string]interface{}, error) {
      v, err := decoder(req)
      if err != nil {
         return v, err
      }
      utility.InsertInMap(v, key, augment(req))
      return v, nil
   }
}
```


#### cli
```py
class Transport(HTTPTransportBase):

    scheme = ['http', 'https']

    def __init__(self, parsed_url, **kwargs):
        super(Transport, self).__init__(parsed_url, **kwargs)
        pool_kwargs = {
            'cert_reqs': 'CERT_REQUIRED',
            'ca_certs': certifi.where(),
            'block': True,
        }
        if not self._verify_server_cert:
            pool_kwargs['cert_reqs'] = ssl.CERT_NONE
            pool_kwargs['assert_hostname'] = False
        proxy_url = os.environ.get('HTTPS_PROXY', os.environ.get('HTTP_PROXY'))
        if proxy_url:
            self.http = urllib3.ProxyManager(proxy_url, **pool_kwargs)
        else:
            self.http = urllib3.PoolManager(**pool_kwargs)

    def send(self, data, headers, timeout=None):
        response = None

        # ensure headers are byte strings
        headers = {k.encode('ascii') if isinstance(k, compat.text_type) else k:
                   v.encode('ascii') if isinstance(v, compat.text_type) else v
                   for k, v in headers.items()}
        if compat.PY2 and isinstance(self._url, compat.text_type):
            url = self._url.encode('utf-8')
        else:
            url = self._url
        try:
            try:
                response = self.http.urlopen(
                    'POST', url, body=data, headers=headers, timeout=timeout, preload_content=False
                )
                logger.info('Sent request, url=%s size=%.2fkb status=%s', url, len(data) / 1024.0, response.status)
            except Exception as e:
                print_trace = True
                if isinstance(e, MaxRetryError) and isinstance(e.reason, TimeoutError):
                    message = (
                        "Connection to APM Server timed out "
                        "(url: %s, timeout: %s seconds)" % (self._url, timeout)
                    )
                    print_trace = False
                else:
                    message = 'Unable to reach APM Server: %s (url: %s)' % (
                        e, self._url
                    )
                raise TransportException(message, data, print_trace=print_trace)
            body = response.read()
            if response.status >= 400:
                if response.status == 429:  # rate-limited
                    message = 'Temporarily rate limited: '
                    print_trace = False
                else:
                    message = 'HTTP %s: ' % response.status
                    print_trace = True
                message += body.decode('utf8')
                raise TransportException(message, data, print_trace=print_trace)
            return response.getheader('Location')
        finally:
            if response:
                response.close()


class AsyncTransport(AsyncHTTPTransportBase, Transport):
    scheme = ['http', 'https']
    async_mode = True
    sync_transport = Transport
```
