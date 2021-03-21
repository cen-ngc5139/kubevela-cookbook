# ApplicationContext

## CRD

```go
// ApplicationContextSpec is the spec of ApplicationContext
type ApplicationContextSpec struct {
   // ApplicationRevisionName points to the snapshot of an Application with all its closure
   ApplicationRevisionName string `json:"applicationRevisionName"`
}

// ApplicationContext is the Schema for the ApplicationContext API
// +kubebuilder:object:root=true
// +kubebuilder:resource:shortName=appcontext,categories={oam}
// +kubebuilder:subresource:status
type ApplicationContext struct {
   metav1.TypeMeta   `json:",inline"`
   metav1.ObjectMeta `json:"metadata,omitempty"`

   Spec ApplicationContextSpec `json:"spec,omitempty"`
   // we need to reuse the AC status
   Status ApplicationConfigurationStatus `json:"status,omitempty"`
}

// ApplicationContextList contains a list of ApplicationContext
// +kubebuilder:object:root=true
type ApplicationContextList struct {
   metav1.TypeMeta `json:",inline"`
   metav1.ListMeta `json:"metadata,omitempty"`
   Items           []ApplicationContext `json:"items"`
}
```

