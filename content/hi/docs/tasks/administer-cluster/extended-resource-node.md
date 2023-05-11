---
title: नोड के लिए विस्तारित रिसोर्सेज का विज्ञापन करें
content_type: task
weight: 70
---

यह पृष्ठ दिखाता है कि किसी नोड के लिए विस्तारित रिसोर्सेज को कैसे निर्दिष्ट किया जाए। विस्तारित रिसोर्स क्लस्टर एडमिनिस्ट्रेटर्स को नोड-स्तरीय रिसोर्सेज का विज्ञापन करने की अनुमति देते हैं जो अन्यथा कुबेरनेट्स के लिए अज्ञात होंगे।


## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

## अपने नोड्स के नाम प्राप्त करें

```shell
kubectl get nodes
```

इस अभ्यास के लिए उपयोग करने के लिए अपना एक नोड चुनें।

## अपने किसी एक नोड पर एक नए विस्तारित रिसोर्स का विज्ञापन करें

नोड पर एक नए विस्तारित रिसोर्स का विज्ञापन करने के लिए, कुबेरनेट्स API सर्वर को एक HTTP PATCH अनुरोध भेजें। उदाहरण के लिए, मान लीजिए कि आपके एक नोड में चार डोंगल जुड़े हुए हैं। यहां PATCH अनुरोध का एक उदाहरण दिया गया है जो आपके नोड के लिए चार डोंगल संसाधनों का विज्ञापन करता है।

```
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: k8s-master:8080

[
  {
    "op": "add",
    "path": "/status/capacity/example.com~1dongle",
    "value": "4"
  }
]
```

ध्यान दें कि कुबेरनेट्स को यह जानने की आवश्यकता नहीं है कि डोंगल क्या है या डोंगल किस लिए है। पूर्ववर्ती PATCH अनुरोध कुबेरनेट्स को बताता है कि आपके नोड में चार चीजें हैं जिन्हें आप डोंगल कहते हैं।

एक प्रॉक्सी प्रारंभ करें, ताकि आप कुबेरनेट्स API सर्वर को आसानी से अनुरोध भेज सकें:

```shell
kubectl proxy
```

अन्य कमांड विंडो में, HTTP PATCH अनुरोध भेजें। `<your-node-name>` को अपने नोड के नाम से बदलें:

```shell
curl --header "Content-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op": "add", "path": "/status/capacity/example.com~1dongle", "value": "4"}]' \
  http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

{{< note >}}
पिछले अनुरोध में, `~1` पैच पथ के वर्ण / के  लिए एन्कोडिंग है। JSON-पैच में ऑपरेशन पथ मान को JSON-पॉइंटर के रूप में समझा जाता है। अधिक जानकारी के लिए देखें [IETF RFC 6901](https://tools.ietf.org/html/rfc6901), सेक्शन 3।
{{< /note >}}

आउटपुट से पता चलता है कि नोड में 4 डोंगल की क्षमता है:

```
"capacity": {
  "cpu": "2",
  "memory": "2049008Ki",
  "example.com/dongle": "4",
```

अपने नोड का वर्णन करें:

```
kubectl describe node <your-node-name>
```

एक बार फिर, आउटपुट डोंगल रिसोर्स दिखाता है:

```yaml
Capacity:
  cpu: 2
  memory: 2049008Ki
  example.com/dongle: 4
```

अब, एप्लिकेशन डेवलपर पॉड्स बना सकते हैं जो एक निश्चित संख्या में डोंगल का अनुरोध करते हैं। [एक कंटेनर को विस्तारित रिसोर्सेज असाइन करें](/docs/tasks/configure-pod-container/extended-resource/) देखें।

## चर्चा

विस्तारित रिसोर्सेज मेमोरी और cpu रिसोर्सेज के समान हैं। उदाहरण के लिए, जिस तरह एक नोड के पास एक निश्चित मात्रा में मेमोरी होती है और नोड पर चल रहे सभी कंपोनेंट्स द्वारा cpu को साझा किया जाता है, उसी तरह नोड पर चलने वाले सभी कंपोनेंट्स द्वारा साझा किए जाने वाले डोंगल की एक निश्चित संख्या हो सकती है। और जिस तरह एप्लिकेशन डेवलपर पॉड्स बना सकते हैं जो एक निश्चित मात्रा में मेमोरी और cpu का अनुरोध करते हैं, वे पॉड्स बना सकते हैं जो एक निश्चित संख्या में डोंगल का अनुरोध करते हैं।

विस्तारित रिसोर्सेज कुबेरनेट्स के लिए अपारदर्शी हैं; कुबेरनेट्स को कुछ भी पता नहीं है कि वे क्या हैं। कुबेरनेट्स केवल यह जानता है कि एक नोड के पास वे निश्चित संख्या मे है। विस्तारित रिसोर्सेज को पूर्णांक मात्रा में विज्ञापित किया जाना चाहिए। उदाहरण के लिए, एक नोड चार डोंगल का विज्ञापन कर सकता है, लेकिन 4.5 डोंगल का नहीं।

### स्टोरेज उदाहरण

मान लीजिए कि एक नोड में 800 GiB विशेष प्रकार का डिस्क स्टोरेज है। आप विशेष उदाहरण के लिए एक नाम बना सकते हैं, जैसे example.com/special-storage। फिर आप इसे एक निश्चित आकार के टुकड़ों में विज्ञापित कर सकते हैं, मान लीजिए 100 GiB। उस स्थिति में, आपका नोड विज्ञापित करेगा कि उसके पास example.com/special-storage प्रकार के आठ रिसोर्सेज हैं।

```yaml
Capacity:
 ...
 example.com/special-storage: 8
```

यदि आप विशेष स्टोरेज के लिए मनमाना अनुरोधों की अनुमति देना चाहते हैं, तो आप 1 बाइट साइज के टुकड़ों में विशेष स्टोरेज का विज्ञापन कर सकते हैं। उस स्थिति में, आप example.com/special-storage प्रकार के 800Gi रिसोर्सेज का विज्ञापन करेंगे।

```yaml
Capacity:
 ...
 example.com/special-storage:  800Gi
```

फिर एक कंटेनर 800Gi तक के विशेष स्टोरेज के किसी भी बाइट्स का अनुरोध कर सकता है।

## क्लीन उप
यहां एक PATCH अनुरोध है जो डोंगल विज्ञापन को नोड से हटाता है।
```
PATCH /api/v1/nodes/<your-node-name>/status HTTP/1.1
Accept: application/json
Content-Type: application/json-patch+json
Host: k8s-master:8080

[
  {
    "op": "remove",
    "path": "/status/capacity/example.com~1dongle",
  }
]
```


एक प्रॉक्सी प्रारंभ करें, ताकि आप कुबेरनेट्स API सर्वर को आसानी से अनुरोध भेज सकें:

```shell
kubectl proxy
```

अन्य कमांड विंडो में, HTTP PATCH अनुरोध भेजें।
`<your-node-name>` को अपने नोड के नाम से बदलें:

```shell
curl --header "Content-Type: application/json-patch+json" \
  --request PATCH \
  --data '[{"op": "remove", "path": "/status/capacity/example.com~1dongle"}]' \
  http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

सत्यापित करें कि डोंगल विज्ञापन हटा दिया गया है:
```
kubectl describe node <your-node-name> | grep dongle
```

(आपको कोई आउटपुट नहीं दिखाना चाहिए)

## {{% heading "whatsnext" %}}

### एप्लिकेशन डेवलपर्स के लिए

- [एक कंटेनर में विस्तारित रिसोर्सेज असाइन करें](/docs/tasks/configure-pod-container/extended-resource/)

### क्लस्टर एडमिनिस्ट्रेटर्स के लिए

- [किसी नेमस्पेस के लिए न्यूनतम और अधिकतम मेमोरी प्रतिबंध कॉन्फ़िगर करें](/docs/tasks/administer-cluster/manage-resources/memory-constraint-namespace/)
- [किसी नेमस्पेस के लिए न्यूनतम और अधिकतम cpu प्रतिबंध कॉन्फ़िगर करें](/docs/tasks/administer-cluster/manage-resources/cpu-constraint-namespace/)